---
layout: post
title:  "Dissecting TCP segments in Wireshark"
date:   2017-02-07
categories: programming wireshark lua
---

Let's say you have a custom application, that is exhibiting strange behavior and
now you want to understand what data exactly is transferred over the network.
Enter: Wireshark!

<p><center><img src="{{site.baseurl}}/assets/blog/pictures/wireshark1.png" style="width:600px;height:auto" alt="A Wireshark dissector"></center></p>

With a vanilla Wireshark you can find the TCP streams or UDP packets of your
apllication, but you will have a hard time to understand whatever data is
contained in the payload. For debugging it would be good to extract this data
and have it represented directly in Wireshark. Fortunately, Wireshark enables
you to do exactly this by writing a custom plugin, called a "dissector", either
in LUA or in C.

If you choose LUA, then I have some tips for you!

First of all a quick summary of the most important LUA resources:

* The [LUA API] which is good to keep at hand, because most of the objects you
  will be handling are either tables, sort of hash tables with a special support
for arrays, or userdata. The latter can be basically any sort of C data
structure.
* The [LUA reference API] is even more detailed.
* The [Example section] gives a good overview of the different type of LUA
  plugins. There are dissectors -- for extracting information from packets, tap
-- for generating statistics over multiple packets and post-dissectors -- which
are run after regular dissectors.
* Consider especially the example [dissector] that is linked on the example
  website. It gives a good overview of the advanced features you might need.

So let's get to it. You should already have an outline of the packet structure.
That is you should know what bytes belong to which field and what type these
fields are. Keep in mind that even if you are aware of the packet structure,
there might be empty spaces between the fields, due to fields only beginning at
multiple of 8 bytes or similar. See for example this post on [packing in C].

Ok so let us start by putting this documentation into LUA:

{% highlight LUA %}
local drstrange = Proto("drstrange","Strange")
drstrange.fields.symbol = ProtoField.uint32('drstrange.symbol', "Symbol", base.HEX)
drstrange.fields.length = ProtoField.uint32('drstrange.length', "Length", base.DEC)
drstrange.fields.content = ProtoField.string('drstrange.content', "Content")
{% endhighlight %}

In this example, we add the [protocol] "Strange" with the shorthand version
"drstrange". (Because, why not.) In the first line we declare the protocol with
`Proto("drstrange","Strange")`. In the following lines we add multiple [ProtoField] to
the protocol. Now we only have to actually use them on the packet:

{% highlight LUA %}
function drstrange.dissector(tvbuf, pinfo, tree)
  local t = tree:add(drstrange, tvbuf, "Strange")
  t:add(drstrange.fields.symbol, tvbuf(0, 4)) -- a unique symbol
  t:add(drstrange.fields.length, tvbuf(4, 8)) -- a length field in 4 bytes
  t:add(drstrange.fields.content, tvbuf(12, len)) -- content as long as length
end
{% endhighlight %}

So what do we have here? We improve our protocol datastructure by implementing a
dissector function. The function takes three arguments, a [buffer] containing the
actual data, a [packet info object] and the protocol [tree]. We add the protocol
to the tree and add the fields one by one.

The final task is to add the protocol to an existing protocol tree in Wireshark. For simplicity let us assume that we want to add it to all traffic sent over port 8123:

{% highlight LUA %}
local tcp_port_table = DissectorTable.get("tcp.port")
tcp_port_table:add(8123, drstrange.dissector)
{% endhighlight %}

With this last piece we are finished. But, only in the simplest case. 

A couple of scenarios might appear that require you to dive deeper:
1. Traffic of your application will appear on a non-predictable set of ports.
2. You might have some chunks of data that are split across multiple TCP segments.

For the first problem you can use heuristics. It helps if your traffic is contains a unique symbol at the beginning. Then you can implement a simple function:

{% highlight LUA %}
function heur_drstrange_fun(tvbuf, pinfo, tree)
  if tvbuf(0,4):uint() == 100 then
    drstrange.dissector(tvbuf, pinfo, tree)
    return true
  end
  return false
end
{% endhighlight %}

And register this function with your protocol:

{% highlight LUA %}
drstrange:register_heuristic("tcp", heur_drstrange_fun)
{% endhighlight %}

The second problem is a bit more complicated. When your data is split over
multiple packets, you have at least two solutions for reassembling your packet.

<p><center><img src="{{site.baseurl}}/assets/blog/pictures/wireshark2.png" style="width:600px;height:auto" alt="A Wireshark dissector with reassembled data"></center></p>

1. If your packet contains a length field in a fixed length part of the header
   you can use the function [dissect_tcp_pdus].

{% highlight LUA %}
function total_length(tvb, pinfo, offset)
  return tvb(0, 4):uint()
end

function drstrange_tcp_pdus(tvbuf, pinfo, tree)
  local len = get_length()
  tree:add(drstrange.fields.content, tvbuf(4, len))
end

function drstrange.dissector(tvbuf, pinfo, tree)
  local t = tree:add(drstrange, tvbuf, "Strange")
  t:add(drstrange.fields.length, tvbuf(0, 4))
  dissect_tcp_pdus(tvbuf, tree, 4, total_length, drstrange_tcp_pdus)
end
{% endhighlight %}

2. If the length of the whole data cannot be obtained from a fixed part of the
   header, then you need to dive a bit deeper into Wireshark. The `pinfo` object
contains an attribute `desegment_len`. This attribute specifies how many bytes
of the data are still missing. A small example:

{% highlight LUA %}
function drstrange.dissector(tvbuf, pinfo, tree)
  local t = tree:add(drstrange, tvbuf, "Strange")
  t:add(drstrange.fields.symbol, tvbuf(0, 4)) -- a unique symbol
  t:add(drstrange.fields.length, tvbuf(4, 8)) -- a length field in 4 bytes
  local missing_data = tvbuf(4, 8):uint() - tvbuf:len()
  if missing_data > 0 then
    pinfo.desegment_len = missing_data
    return
  end
  t:add(drstrange.fields.content, tvbuf(12, len)) -- content as long as length
end
{% endhighlight %}

Wireshark will keep on calling the dissector with increasingly big tvbuf
objects, till either the requirement is satisfied or no more data is available.
This behavior seems to be documented nowhere, except for some blog posts and
example code here and there.

A final word of caution. If your packet trace misses out some packets, the
whole TCP reassembly process can be easily broken. You want to ensure this does
not happen.

If I find the time, I will discuss this topic in a later post. So far from me :).

[LUA API]: https://wiki.wireshark.org/LuaAPI
[LUA reference API]: https://www.wireshark.org/docs/wsdg_html/#wsluarm_modules
[dissect_tcp_pdus]: https://www.wireshark.org/docs/wsdg_html/#global_functions_Proto
[Example section]: https://wiki.wireshark.org/Lua/Examples
[dissector]: https://wiki.wireshark.org/Lua/Examples?action=AttachFile&do=get&target=dissector.lua
[packing in C]: http://www.catb.org/esr/structure-packing/
[ProtoField]: https://wiki.wireshark.org/LuaAPI/Proto#ProtoField
[protocol]: https://wiki.wireshark.org/LuaAPI/Proto#Proto
[buffer]: https://wiki.wireshark.org/LuaAPI/Tvb#Tvb
[packet info object]: https://wiki.wireshark.org/LuaAPI/Pinfo
[tree]: https://wiki.wireshark.org/LuaAPI/TreeItem
