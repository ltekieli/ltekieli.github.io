---
layout: post
title: Setting up Bazel
date: '2025-01-03 19:01:00'
---

# Setting Up Bazel for C++ Development

Now that you’ve been introduced to Bazel and its advantages, it’s time to get hands-on. In this post, we’ll walk through setting up Bazel for your C++ development environment and building your first project. By the end, you’ll have a fully functional Bazel setup ready for future projects.

## Step 1: Installing Bazel

To install Bazel, refer to the [official Bazel documentation](https://bazel.build/) for platform-specific instructions. The documentation provides detailed steps for Linux, macOS, and Windows environments.

For version management, it’s highly recommended to use **Bazelisk**, a wrapper for Bazel that automatically downloads and uses the appropriate Bazel version for your project. You can download Bazelisk from the [official Bazelisk releases page](https://github.com/bazelbuild/bazelisk/releases) and follow the provided installation instructions.

Using Bazelisk ensures your builds are consistent and compatible with the required Bazel version.

## Step 2: Verifying the Installation

After installation, verify that Bazel is correctly installed by running:
```bash
bazel version
```
You should see the installed Bazel version and its build details.

## Step 3: Setting Up a C++ Project with Bazel

Let’s create a simple "Hello, World!" project to test your Bazel setup.

### Create the Project Directory Structure
```plaintext
hello_bazel/
├── main.cpp
└── BUILD
```

### Write the C++ Source File
Create a `main.cpp` file with the following content:
```cpp
#include <iostream>

int main() {
    std::cout << "Hello, Bazel!" << std::endl;
    return 0;
}
```

### Define the BUILD File
Create a `BUILD` file with the following content:
```python
cc_binary(
    name = "hello_bazel",
    srcs = ["main.cpp"],
)
```
This tells Bazel to create a C++ binary named `hello_bazel` using `main.cpp`.

## Step 4: Building and Running the Project

### Build the Project
Run the following command in the project directory:
```bash
bazel build //:hello_bazel
```
Bazel will generate the binary in the `bazel-bin` directory.

### Run the Binary
Use Bazel to run the binary directly:
```bash
bazel run //:hello_bazel
```
You should see:
```plaintext
Hello, Bazel!
```

## What’s Next?

Check the article list in the [introduction post](/what-is-bazel-and-why-use-it/#summary)
