
./tensorflow/lite/tools/build_aar.sh \
  --target_archs=arm64-v8a,armeabi-v7a \
  --custom_ops_register_file=/tmp/custom_ops.cc \
  --extra_tflite_flex_ops=SentencepieceOp \
  --extra_bazel_args="--copt=-DGETPAGESIZE_FORCE=16384 --linkopt=-Wl,-z,max-page-size=16384"
  
Of course. Building a custom `select-ops.aar` for a specific model is an excellent way to ensure compatibility while optimizing for size.

This guide consolidates all the steps, including the necessary fixes for environment setup, into a single workflow.

-----

### \#\# Overview

The process involves four main stages performed from your Ubuntu machine:

1.  **Setup a Docker Environment**: We'll use Docker to create a clean and consistent build environment.
2.  **Install Android Tools**: Inside Docker, we'll install the specific SDK and NDK needed for the build.
3.  **Generate Ops List**: We will analyze your model to create a list of the exact operators it needs.
4.  **Build the AAR**: We'll compile a minimal AAR using the generated operator list.

-----

### \#\# Step 1: Prepare the Build Environment

1.  **Install Prerequisites**: Make sure you have Git and Docker installed on your Ubuntu machine.
    ```bash
    sudo apt update
    sudo apt install git docker.io
    sudo usermod -aG docker $USER 
    # Log out and log back in to apply the docker group change
    ```
2.  **Clone TensorFlow**: Clone the repository and check out a specific stable version.
    ```bash
    git clone https://github.com/tensorflow/tensorflow.git
    cd tensorflow
    git checkout v2.16.1
    ```
3.  **Place Your Model**: Copy your `.tflite` model file into the `tensorflow` directory you just cloned. This will make it accessible inside the Docker container.
4.  **Start the Docker Container**: This command starts the container and creates a shared folder with your `tensorflow` directory.
    ```bash
    docker run -it -w /tensorflow -v $PWD:/tensorflow tensorflow/build:latest-python3.9
    ```
    You are now inside the Docker container. All subsequent commands are run here.

-----

### \#\# Step 2: Install Android SDK & NDK

The default build container is missing the Android tools. Run the following commands inside the container to install them.

1.  **Install Tools & Android NDK**:
    ```bash
    apt-get update && apt-get install -y wget unzip
    cd /
    wget https://dl.google.com/android/repository/android-ndk-r25c-linux.zip
    unzip android-ndk-r25c-linux.zip
    rm android-ndk-r25c-linux.zip
    ```
2.  **Install Android SDK**:
    ```bash
    mkdir -p /android/sdk && cd /android/sdk
    wget https://dl.google.com/android/repository/commandlinetools-linux-9477386_latest.zip
    unzip commandlinetools-linux-9477386_latest.zip
    mkdir -p cmdline-tools/latest
    mv cmdline-tools/* cmdline-tools/latest/
    rm commandlinetools-linux-9477386_latest.zip
    yes | cmdline-tools/latest/bin/sdkmanager --licenses
    cmdline-tools/latest/bin/sdkmanager "platforms;android-33" "build-tools;33.0.2" "platform-tools"
    ```
3.  **Return to the Workspace**:
    ```bash
    cd /tensorflow
    ```

-----

### \#\# Step 3: Configure the Build

Now, configure the TensorFlow workspace to use the tools you just installed.

1.  Run the configure script:
    ```bash
    ./configure
    ```
2.  Answer the prompts as follows:
      * **Python Location**: Press **Enter** to accept the default.
      * **ROCm and CUDA Support**: Type **N** for both and press **Enter**.
      * **Android SDK Path**: Manually type `/android/sdk` and press **Enter**.
      * **Android NDK Path**: Manually type `/android-ndk-r25c` and press **Enter**.
      * For all other questions, press **Enter** to accept the defaults.

-----

### \#\# Step 4: Build the Selective AAR 📦

Now we'll perform the two-step selective build process.

1.  **Generate Operator List**: Run the analysis script on your model. **Replace `your_model.tflite` with the actual name of your file.**
    ```bash
    python3 tensorflow/lite/tools/tflite_custom_ops_list.py \
      --model_file=/tensorflow/your_model.tflite \
      --output_file=/tmp/custom_ops.cc
    ```
2.  **Build the AAR**: Run the build script, pointing it to your generated operator list.
    ```bash
    ./tensorflow/lite/tools/build_aar.sh \
      --target_archs=arm64-v8a,armeabi-v7a \
      --custom_ops_register_file=/tmp/custom_ops.cc
    ```

The build process will now begin and may take a significant amount of time.

-----

### \#\# Step 5: Access Your AAR ✅

1.  **Location**: The final, optimized library will be located at:
    `tensorflow/lite/java/bazel-bin/tensorflow-lite.aar`

2.  **Accessing the File**: The file is **already on your main Ubuntu machine** because of the shared folder.

3.  **Fixing Permissions**: The build runs as `root` in the container, so you must fix the file ownership on your host machine. **Open a new Ubuntu terminal** (not the Docker one), navigate to your `tensorflow` directory, and run:

    ```bash
    sudo chown -R -L $USER:$USER bazel-bin
    ```

You now have your custom-built, minimal `.aar` file ready to be included in your Android Studio project.
