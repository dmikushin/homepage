---
layout: post
title: "Thrust/CUDA tip: reuse temporary buffer across multiple transforms"
tags:
  - Software Engineering
thumbnail_path: blog/2014-10-09-thrust-cuda-tip-reuse-temporary-buffer-across-multiple-transforms/thrust_logo.png
---

[Thrust](thrust.github.io) is a very handy STL-like template library for rapid data processing on GPUs.

![alt text](\assets\img\blog\2014-10-09-thrust-cuda-tip-reuse-temporary-buffer-across-multiple-transforms\thrust_logo.png "Logo Title Text 1")

In real applications it is often needed to perform the same data processing (in Thrust's terminology - transform) multiple times on different datasets. Transforms containing reduction (reduction, sorting, transform-reduce, etc.) require temporary arrays allocation. By default these allocations are performed for each individual transform, adding cudaMalloc/cudaFree operations, that could be quite expensive

![alt text](\assets\img\blog\2014-10-09-thrust-cuda-tip-reuse-temporary-buffer-across-multiple-transforms\usual_call.png "Logo Title Text 1")

An obvious optimization method here would be to reuse single temporary arrays set across multiple transforms. This could be done by hooking cudaMalloc/cudaFree at application level. More elegant solution exists in Thrust, recently pointed out by Eli Graser:

```
#include <thrust/system/cuda/vector.h>
#include <thrust/system/cuda/execution_policy.h>
#include <thrust/host_vector.h>
#include <thrust/generate.h>
#include <thrust/pair.h>
#include <iostream>
#include <map>

// Example by Nathan Bell and Jared Hoberock
// (modified by Mihail Ivakhnenko)
//
// This example demonstrates how to intercept calls to get_temporary_buffer
// and return_temporary_buffer to control how Thrust allocates temporary storage
// during algorithms such as thrust::reduce. The idea will be to create a simple
// cache of allocations to search when temporary storage is requested. If a hit
// is found in the cache, we quickly return the cached allocation instead of
// resorting to the more expensive thrust::cuda::malloc.
//
// Note: this implementation cached_allocator is not thread-safe. If multiple
// (host) threads use the same cached_allocator then they should gain exclusive
// access to the allocator before accessing its methods.

// cached_allocator: a simple allocator for caching allocation requests
class cached_allocator
{
public:
    // just allocate bytes
    typedef char value_type;

    cached_allocator() { }

    ~cached_allocator()
    {
        // free all allocations when cached_allocator goes out of scope
        free_all();
    }

    char* allocate(std::ptrdiff_t num_bytes)
    {
        char* result = 0;

        // search the cache for a free block
        free_blocks_type::iterator free_block = free_blocks.find(num_bytes);

        if (free_block != free_blocks.end())
        {
            std::cout << "cached_allocator::allocator(): found a hit" << std::endl;

            // get the pointer
            result = free_block->second;

            // erase from the free_blocks map
            free_blocks.erase(free_block);
        }
        else
        {
            // no allocation of the right size exists
            // create a new one with cuda::malloc
            // throw if cuda::malloc can't satisfy the request
            try
            {
                std::cout << "cached_allocator::allocator(): no free block found; calling cuda::malloc" << std::endl;

                // allocate memory and convert cuda::pointer to raw pointer
                result = thrust::cuda::malloc<char>(num_bytes).get();
            }
            catch(std::runtime_error &e)
            {
                throw;
            }
        }

        // insert the allocated pointer into the allocated_blocks map
        allocated_blocks.insert(std::make_pair(result, num_bytes));

        return result;
    }

    void deallocate(char* ptr, size_t n)
    {
        // erase the allocated block from the allocated blocks map
        allocated_blocks_type::iterator iter = allocated_blocks.find(ptr);
        std::ptrdiff_t num_bytes = iter->second;
        allocated_blocks.erase(iter);

        // insert the block into the free blocks map
        free_blocks.insert(std::make_pair(num_bytes, ptr));
    }

private:
    typedef std::multimap<std::ptrdiff_t, char*> free_blocks_type;
    typedef std::map<char*, std::ptrdiff_t> allocated_blocks_type;

    free_blocks_type free_blocks;
    allocated_blocks_type allocated_blocks;

    void free_all()
    {
        std::cout << "cached_allocator::free_all(): cleaning up after ourselves..." << std::endl;

        // deallocate all outstanding blocks in both lists
        for (free_blocks_type::iterator i = free_blocks.begin();
                i != free_blocks.end(); i++)
        {
            // transform the pointer to cuda::pointer before calling cuda::free
            thrust::cuda::free(thrust::cuda::pointer<char>(i->second));
        }

        for (allocated_blocks_type::iterator i = allocated_blocks.begin();
                i != allocated_blocks.end(); i++)
        {
            // transform the pointer to cuda::pointer before calling cuda::free
            thrust::cuda::free(thrust::cuda::pointer<char>(i->first));
        }
    }
};

int main()
{
#if defined(THRUST_GCC_VERSION) && (THRUST_GCC_VERSION < 40400)
    std::cout << "This feature requires gcc >= 4.4" << std::endl;
    return 0;
#endif

    size_t n = 1 << 22;

    // create a cached_allocator object
    cached_allocator alloc;

    // generate and reduce random input
    thrust::host_vector<int> a_r_input(n);
    thrust::generate(a_r_input.begin(), a_r_input.end(), rand);
    thrust::cuda::vector<int> a_input = a_r_input;
    thrust::reduce(thrust::cuda::par(alloc), a_input.begin(), a_input.end());

    // generate and reduce random input
    thrust::host_vector<int> b_r_input(n);
    thrust::generate(b_r_input.begin(), b_r_input.end(), rand);
    thrust::cuda::vector<int> b_input = b_r_input;
    thrust::reduce(thrust::cuda::par(alloc), b_input.begin(), b_input.end());

    // generate and reduce random input
    thrust::host_vector<int> c_r_input(n);
    thrust::generate(c_r_input.begin(), c_r_input.end(), rand);
    thrust::cuda::vector<int> c_input = c_r_input;
    thrust::reduce(thrust::cuda::par(alloc), c_input.begin(), c_input.end());

    // generate and reduce random input
    thrust::host_vector<int> d_r_input(n);
    thrust::generate(d_r_input.begin(), d_r_input.end(), rand);
    thrust::cuda::vector<int> d_input = d_r_input;
    thrust::reduce(thrust::cuda::par(alloc), d_input.begin(), d_input.end());

    // generate and reduce random input
    thrust::host_vector<int> e_r_input(n);
    thrust::generate(e_r_input.begin(), e_r_input.end(), rand);
    thrust::cuda::vector<int> e_input = e_r_input;
    thrust::reduce(thrust::cuda::par(alloc), e_input.begin(), e_input.end());

    return 0;
}
```

With this custom allocator, cudaMalloc and cudaFree are now called only once, no additional calls between reductions:

![alt text](\assets\img\blog\2014-10-09-thrust-cuda-tip-reuse-temporary-buffer-across-multiple-transforms\changed_call.png "Logo Title Text 1")

In the real financial application we develop this optimization resulted into 8% performance improvement.
