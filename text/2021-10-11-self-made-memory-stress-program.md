# Self-made Memory Stress Program

## Summary

Create a program to replace stress-ng to realize mem stress.

## Motivation

Because stress-ng realize memory stress by applying read and write pressure instead of occupying memory space, which will affect CPU performance, a program purely takes up memory space is need to generate a memory StressChaos.

## Detailed design

Using `syscall.Mmap()` to allocate memory space directly, so that uncertain footprint can be avoided. And write 1 byte on every virtual memory page of the space we allocated can achieve our goal.

```data, err := syscall.Mmap(-1, 0, length, syscall.PROT_READ|syscall.PROT_WRITE, syscall.MAP_PRIVATE|syscall.MAP_ANONYMOUS)```

Furthermore, users can specify time to let the memory allocation increase linearly, so they can simulate a memory leak. The program will write certain numbers of pages in a time interval longer than 100ms , which can reduce the impact of the delay of `time.Sleep()`.

## Alternatives

- stress-ng will obviously increase the usage of CPU.
- Allocating memory space by using `make()` will cause uncertain footprint. And GC may be triggered under certain situation.

## Unresolved questions

- The program itself takes about 68MB
- It takes time to allocate big memory (over a giga bytes) without the parameter `--time`. For example, it will take 3.5s to allocate 12GB memory space.
