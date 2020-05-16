# Chapter 3 - Connecting to a deployment
## Code branch

https://github.com/improbable-andreaskrugersen/spatialstein3d/tree/chapter3-client-deployment

## Goals

Last chapter we created a basic SpatialOS project setup and we can now build our project using `spatial build`. In this chapter we will start using the Worker SDK to connect to a deployment and turn our code into a real client-worker.

## Choice of Worker SDK language

Since the project was written in C++ so far, we have two choices which Worker SDK to use:
- [Worker SDK in C++](https://documentation.improbable.io/sdks-and-data/docs/cpp-introduction)
- [Worker SDK in C](https://documentation.improbable.io/sdks-and-data/docs/c-introduction)

The C++ SDK is more geared towards evaluation than to be used in a production game but it has the advantage of code generation for our schema files. Since this project is not supposed to be a production-ready game and we want to get up and running quickly, we'll use the C++ API. Feel free to read the introduction pages linked above to get more information about the trade-offs of using either one.

