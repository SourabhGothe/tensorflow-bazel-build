Perfect — so you want to **use Bazel directly (instead of `build_aar.sh`)** and still give it an **input `.tflite` model** so it builds a **selective TensorFlow Lite AAR** (only with ops used in that model, plus your custom ops like `SentenceOp`) — and you want this build also to respect the **16 KB page alignment**.

Let’s go step-by-step with the **exact Bazel command**.

---

## 🧠 Background

When you run:

`sh tensorflow/lite/tools/build_aar.sh --input_models=...`

that script internally calls a Bazel target:

`//tensorflow/lite/java:custom_ops_aar`

and passes `--define tflite_select_tf_ops=true` and some flags like:

`--define=lite_with_select_tf_ops=true --define=tflite_with_xnnpack=true --define=build_with_select_ops=true --define=use_selective_registration=true --define=selective_registration_targets=<generated from model>`

The “selective registration” part is built using a **generated file** that lists ops extracted from the `.tflite` model(s).  
To do that manually, we first run the model analyzer tool.

---

## 🧩 Step 1: Generate the op list from your model

You need to produce a C++ file or list of ops used in your `.tflite` model.

Run:

`bazel run //tensorflow/lite/tools:optimize_for_mobile -- \   --input_model=/path/to/your_model.tflite \   --output_model=/tmp/optimized.tflite`

Then run:

`bazel run //tensorflow/lite/tools:print_selective_registration -- \   --input_model=/tmp/optimized.tflite \   --output_file=/tmp/op_reg.cc`

This produces `/tmp/op_reg.cc` (the list of used ops).

---

## 🧩 Step 2: Build the AAR with selective registration + extra ops

Now invoke Bazel with:

`bazel build \   --config=android \   --cpu=arm64-v8a \   --define=use_selective_registration=true \   --define=selective_registration_file=/tmp/op_reg.cc \   --define=extra_op_names=SentenceOp \   --linkopt=-Wl,-z,max-page-size=16384 \   --host_linkopt=-Wl,-z,max-page-size=16384 \   //tensorflow/lite/java:tensorflow-lite`

That will:

- Build the **TensorFlow Lite AAR** containing only the ops used in your model.
    
- Force-include the `SentenceOp`.
    
- Use **16 KB alignment** linker flags.
    

You can build both ABIs at once like:

`bazel build \   --config=android \   --fat_apk_cpu=arm64-v8a,armeabi-v7a \   --define=use_selective_registration=true \   --define=selective_registration_file=/tmp/op_reg.cc \   --define=extra_op_names=SentenceOp \   --linkopt=-Wl,-z,max-page-size=16384 \   --host_linkopt=-Wl,-z,max-page-size=16384 \   //tensorflow/lite/java:tensorflow-lite`

---

## 📦 Output

The resulting `.aar` will be under:

`bazel-bin/tensorflow/lite/java/tensorflow-lite.aar`

You can unzip it and check:

`unzip -l bazel-bin/tensorflow/lite/java/tensorflow-lite.aar unzip -p bazel-bin/tensorflow/lite/java/tensorflow-lite.aar jni/arm64-v8a/libtensorflowlite_jni.so > libtflite.so readelf -l libtflite.so | grep p_align strings libtflite.so | grep -i sentence`

You should see:

- `p_align: 0x4000` (16 KB)
    
- Symbols related to `SentenceOp`
    

---

## ⚙️ Summary

|Step|Purpose|Command|
|---|---|---|
|Generate optimized model|Optional, but cleaner|`bazel run //tensorflow/lite/tools:optimize_for_mobile -- --input_model=… --output_model=…`|
|Generate op list|From .tflite|`bazel run //tensorflow/lite/tools:print_selective_registration -- --input_model=… --output_file=/tmp/op_reg.cc`|
|Build AAR|With extra op + 16 KB alignment|`bazel build --config=android … //tensorflow/lite/java:tensorflow-lite`|

---

Would you like me to show the **exact minimal version** of this command that works on a clean TensorFlow checkout (without manually running `print_selective_registration`)?  
(i.e. a single Bazel line that handles everything end-to-end, similar to how `build_aar.sh` does it internally)
