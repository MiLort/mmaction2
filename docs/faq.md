# FAQ

We list some common issues faced by many users and their corresponding solutions here.
Feel free to enrich the list if you find any frequent issues and have ways to help others to solve them.
If the contents here do not cover your issue, please create an issue using the [provided templates](/.github/ISSUE_TEMPLATE/error-report.md) and make sure you fill in all required information in the template.

## Installation

- **"No module named 'mmcv.ops'"; "No module named 'mmcv._ext'"**

    1. Uninstall existing mmcv in the environment using `pip uninstall mmcv`.
    2. Install mmcv-full following the [installation instruction](https://mmcv.readthedocs.io/en/latest/#installation).

- **"OSError: MoviePy Error: creation of None failed because of the following error"**

    Refer to [install.md](https://github.com/open-mmlab/mmaction2/blob/master/docs/install.md#requirements)
    1. For Windows users, [ImageMagick](https://www.imagemagick.org/script/index.php) will not be automatically detected by MoviePy,
    there is a need to modify `moviepy/config_defaults.py` file by providing the path to the ImageMagick binary called `magick`,
    like `IMAGEMAGICK_BINARY = "C:\\Program Files\\ImageMagick_VERSION\\magick.exe"`
    2. For Linux users, there is a need to modify the `/etc/ImageMagick-6/policy.xml` file by commenting out
    `<policy domain="path" rights="none" pattern="@*" />` to `<!-- <policy domain="path" rights="none" pattern="@*" /> -->`,
    if ImageMagick is not detected by moviepy.

## Data

- **FileNotFound like `No such file or directory: xxx/xxx/img_00300.jpg`**

    In our repo, we set `start_index=1` as default value for rawframe dataset, and `start_index=0` as default value for video dataset.
    If users encounter FileNotFound error for the first or last frame of the data, there is a need to check the files begin with offset 0 or 1,
    that is `xxx_00000.jpg` or `xxx_00001.jpg`, and then change the `start_index` value of data pipeline in configs.

- **How should we preprocess the videos in the dataset? Resizing them to a fix size(all videos with the same height-width ratio) like `340x256`(1) or resizing them so that the short edges of all videos are of the same length (256px or 320px)**

    We have tried both preprocessing approaches and found (2) is a better solution in general, so we use (2) with short edge length 256px as the default preprocessing setting. We benchmarked these preprocessing approaches and you may find the results in [TSN Data Benchmark](https://github.com/open-mmlab/mmaction2/tree/master/configs/recognition/tsn) and [SlowOnly Data Benchmark](https://github.com/open-mmlab/mmaction2/tree/master/configs/recognition/tsn).

- **Mismatched data pipeline items lead to errors like `KeyError: 'total_frames'`**

    We have both pipeline for processing videos and frames.

    **For videos**, We should decode them on the fly in the pipeline, so pairs like `DecordInit & DecordDecode`, `OpenCVInit & OpenCVDecode`, `PyAVInit & PyAVDecode` should be used for this case like [this example](https://github.com/open-mmlab/mmaction2/blob/023777cfd26bb175f85d78c455f6869673e0aa09/configs/recognition/slowfast/slowfast_r50_video_4x16x1_256e_kinetics400_rgb.py#L47-L49).

    **For Frames**, the image has been decoded offline, so pipeline item `RawFrameDecode` should be used for this case like [this example](https://github.com/open-mmlab/mmaction2/blob/023777cfd26bb175f85d78c455f6869673e0aa09/configs/recognition/slowfast/slowfast_r50_8x8x1_256e_kinetics400_rgb.py#L49).

    `KeyError: 'total_frames'` is caused by incorrectly using `RawFrameDecode` step for videos, since when the input is a video, it can not get the `total_frame` beforehand.

## Training

- **How to just use trained recognizer models for backbone pre-training ?**

    Refer to [Use Pre-Trained Model](https://github.com/open-mmlab/mmaction2/blob/master/docs/tutorials/2_finetune.md#use-pre-trained-model),
    in order to use the pre-trained model for the whole network, the new config adds the link of pre-trained models in the `load_from`.

    And to use backbone for pre-training, you can change `pretrained` value in the backbone dict of config files to the checkpoint path / url.
    When training, the unexpected keys will be ignored.

- **How to visualize the training accuracy/loss curves in real-time ?**

    Use `TensorboardLoggerHook` in `log_config` like

    ```python
    log_config=dict(interval=20, hooks=[dict(type='TensorboardLoggerHook')])
    ```

    You can refer to [tutorials/1_config.md](tutorials/1_config.md), [tutorials/7_customize_runtime.md](tutorials/7_customize_runtime.md#log-config), and [this](https://github.com/open-mmlab/mmaction2/blob/master/configs/recognition/tsm/tsm_r50_1x1x8_50e_kinetics400_rgb.py#L118).

## Testing

- **How to make predicted score normalized by softmax within [0, 1] ?**

    change this in the config, make `test_cfg = dict(average_clips='prob')`.

- **What if the model is too large and the GPU memory can not fit even only one testing sample ?**

    By default, the 3d models are tested with 10clips x 3crops, which are 30 views in total. For extremely large models, the GPU memory can not fit even only one testing sample (cuz there are 30 views). To handle this, you can set `max_testing_views=n` in test_cfg of the config file. If so, n views will be used as a batch during forwarding to save GPU memory used.

- **How to show test results ?**

    During testing, we can use the command `--out xxx.json/pkl/yaml` to output result files for checking. The testing output has exactly the same order as the test dataset.
    Besides, we provide an analysis tool for evaluating a model using the output result files in [`tools/analysis/eval_metric.py`](/tools/analysis/eval_metric.py)

## Deploying

- **Why is the onnx model converted by mmaction2 throwing error when converting to other frameworks such as TensorRT?**

    For now, we can only make sure that models in mmaction2 are onnx-compatible. However, some operations in onnx may be unsupported by your target framework for deployment, e.g. TensorRT in [this issue](https://github.com/open-mmlab/mmaction2/issues/414). When such situation occurs, we suggest you raise an issue in the repo of your target framework as long as `pytorch2onnx.py` works well and is verified numerically.
