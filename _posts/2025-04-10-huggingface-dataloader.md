---
layout: post
title: "When Test Sets Collide: Hidden Behaviors of the Huggingface Datasets Library"
date: 2025-04-10
categories: huggingface datasets splits python library
---

Recently, I noticed some weirdly bad evaluation results of certain Phi 3.5 Finetuning runs I was working on for a customer.
The strange thing was that previous eval runs (using a different dataset) showed promising performance, while the new (improved) dataset showed ~10% worse raw accuracy. And this was the case only on the test, but not the validation set. What was happening here?

tl;dr: After some debugging, I found out that the evaluation was run on test samples that should not have been included - how did that happen?

Turns out, the [hugginface datasets]() library's [load_dataset()](https://huggingface.co/docs/datasets/v3.5.1/en/package_reference/loading_methods#datasets.load_dataset) function by default uses some kind of undocumented pattern matching¹ to associate folders to data splits when the source images are structured according to the [imagefolder format](https://huggingface.co/docs/datasets/image_load#imagefolder).
This is pretty non-obvious and potentially dangerous if you ask me.
For example, on a folder like this
```
data
├── test
│   ├── image1.png
│   └── image2.png
├── test_scaled
│   ├── image1.png
│   └── image2.png
├── train
│   ├── image1.png
│   └── image2.png
└── validation
    ├── image1.png
    └── image2.png
```

I would have expected 4 splits in the resulting dataset according to the 4 folders `test, test_scaled, train, validation`.
But calling `load_dataset("imagefolder", data_dir="data")`will produce a dataset looking like this:

```
uv run --with datasets --with pillow python

>>> import datasets
>>> datasets.load_dataset("imagefolder", data_dir="data")
>>> Generating train split: 2 examples [00:00, 1017.29 examples/s]
Generating validation split: 2 examples [00:00, 1803.61 examples/s]
Generating test split: 4 examples [00:00, 3854.17 examples/s]
DatasetDict({
    train: Dataset({
        features: ['image', 'label'],
        num_rows: 2
    })
    validation: Dataset({
        features: ['image', 'label'],
        num_rows: 2
    })
    test: Dataset({
        features: ['image', 'label'],
        num_rows: 4
    })
})
```

It contains 3 splits and a **test split that contains both images from the `test` and `test_scaled_images` folder!**
This even happens if the string "test" is neither a prefix nor postfix but somewhere in the middle of the foldername, e.g. `scaled_test_images`.

I would have much preferred the datasets library to follow a [python Zen](https://peps.python.org/pep-0020/) "do the obvious thing" way here of either producing splits based on the folder names or simply ignoring folders with names not matching the commonly used split names. This would have saved me from having to debug the library to find out it's actual behavior (or having to dive deeply into it's source code).

---
1) I still haven't found the exact location of the code producing this behavior. If you do, let me know ;)