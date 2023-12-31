# DJL - PyTorch native Library

## Introduction
This project builds the JNI layer for Java to call PyTorch C++ APIs.
You can find more information in the `src`.

## Prerequisite
You need to install `cmake` and C++ compiler on your machine in order to build

### Linux

```sh
apt-get install -y locales cmake curl unzip software-properties-common
```

## CPU Build

Use the following task to build PyTorch JNI library:

### Mac/Linux

```sh
./gradlew compileJNI
```

### Windows

```cmd
gradlew compileJNI
```

This task will send a Jni library copy to `pytorch-engine` model to test locally.

## GPU build
Note: PyTorch C++ library requires CUDA path set in the system.

Use the following task to build pytorch JNI library for GPU:

### Mac/Linux

```sh
# compile CUDA 11.X version of JNI
./gradlew compileJNI -Pcu11
```

## Windows

```cmd
# compile CUDA 11.X version of JNI
gradlew compileJNI -Pcu11
```

### Format C++ code
It uses clang-format to format the code.

```sh
./gradlew formatCpp
```

## PyTorch native package release

### Step 1: Build new JNI on top of new libtorch on osx, linux-cpu, linux-gpu, windows

1. Spin up a EC2 instance for linux, linux-gpu, windows, windows-gpu and cd pytorch/pytorch-native.
2. Run ./gradlew compileJNI and resolve all the issues you are facing.
3. Raise a PR for the JNI code change and don’t merge it until we have the rest things ready.

### Step 2: Check dependencies of each JNI

1. check the dependencies of each JNI library by `otool -L libdjl_torch.dylib` for osx, `ldd libdjl_torch.so` for linux and `dumpbin /dependents libdjl_torch.dll` for windows.
2. Compare all dependency libraries to those in downloadPyTorchNativeLib, if we miss copying new dependencies, correct the script.
3. modify the version to desired release version in pytorch/pytorch-native/build.gradle and make sure the URL in the task downloadPyTorchNativeLib point to the right, available URL. Usually the URL that is not for the latest version will have %2Bcpu/cuXXX in the end.
4. Make corresponding change on build.sh  and build.cmd
5. Raise PR for script change and get them merge

### Step 3: Upload the new version of libtorch dependencies to S3

1. Spin up a EC2 instance and `cd pytorch/pytorch-native && ./gradlew dPTNL`
2. `cd build/native/lib` and gzip all dependencies (`gzip -r download`)
3. Create a new pytorch-X.X.X in ai.djl/publish bucket with djl-prod account.
4. `aws s3 sync build/native/lib s3://djl-ai/publish/pytorch-X.X.X`

### Step 4: Build new JNI and upload to S3: 

1. Merge the JNI code change.
2. Now every script should point to new PyTorch version except integration and example are still using old pytorch-native version
3. Trigger Native JNI S3 PyTorch and resolve issues if any

### Step 5: Build pytorch-native snapshot

1. Trigger Native Snapshot PyTorch
2. Raise a PR to bump up all PyTorch version to new version and add -SNAPSHOT
3. Test integration test , example and pytorch-engine unit test with snapshot pytorch-native

### Step 6: Publish pytorch-native to staging

1. Trigger Native Release PyTorch
2. Test integration test, example and pytorch-engine unit test with staging pytorch-native
3. Publish to sonatype 
4. Raise a PR to remove all the -SNAPSHOT

## A quick start on implementing pytorch feature with JNI
This is adjusted from the comments in [PR#2349](https://github.com/deepjavalibrary/djl/issues/2349#issuecomment-1409003379).

To implement a simple pytorch feature, generally you can do the following steps.

1. Find the c-api in torch library for the feature to add. This can be done by searching in the document like [this](https://pytorch.org/cppdocs/api/function_namespaceat_1a854b1b19549a17f87a69b5f6b1134e22.html?highlight=bmm) or searching in the torch cpp source code.
2. Implement the JNI and api's in Java. The JNI can then be compiled with gradle commands. Here is the commands you can use on cpu machine, to compile JNI and run it with java api.

    ```sh
    cd engines/pytorch/pytorch-native
    ./gradlew cleanJNI compileJNI
   
    ./gradlew :engines:pytorch:pytorch-jni:clean
    ./gradlew :engines:pytorch:pytorch-engine:test
    ```
    
    **Note**:
    In case your need test with GPU, the compilation needs to be the following:
    
    ```sh
    ./gradlew cleanJNI 
    ./gradlew compileJNI -Pcu11
    ```

3. Document the api and add the unit tests.
4. Format the code and pass the PR tests

    Run the following tasks
    
    ```sh
    ./gradlew fJ fC checkstyleMain checkstyleTest pmdMain pmdTest
    ./gradlew test
    ```

