One single powerful machine, 
Moore's Law has ended, 
Easier and cheaper to build multiple commodity machines, instead of single
high-performance machines
Writing parallel programs is hard, concurrent data access and synchronization

http://large.stanford.edu/courses/2012/ph250/kumar1/

Several Paradigms and frameworks implementing these paradigms have evolved

I want to highlight two different architectures. Both assume that computation
should be done stateless i.e. be a pure computation of the input. In so far the
computation performed over multiple machines 

microservice architecture
- consists of independent single-purpose services interconnected by APIs
- computation is started by the introduction of independent request into the system
- the form of the computation is defined by the content of the request
- the behavior of the overall system is emerging as a result of the interactions
  between the services

dataflow architecture
- consists of independent operators
- operators consist of an inner part, the user-defined function, and an outer
  shell, that is the form of the input and the output and the in and out-degree.
  (Also described as contracts)
-- the former: map vs flatMap
-- the latter: map vs reduce
- data flow are forwarded between these units
- no synchronization
- central manager
- Extension to Dataflow: 
-- Windowing: Time and Count Windows, parallel windowing requires synchronization
-- Iteration: 
- Computation follows the functional programming paradigm, i.e. all operators
  are pure functions, which perform computation only based on the inputs
-- It is possible to break out of this model, but this can lead to unpredictable
   behavior

Bulk Synchronous Programming (BSP)
Stale Synchronous Programming (SSP)
