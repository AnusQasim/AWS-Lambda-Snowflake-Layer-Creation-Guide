# AWS Lambda Snowflake Layer Creation Guide

## Overview

This document explains how to create a Lambda Layer for the Snowflake Python Connector.

A Lambda Layer is used to store external Python libraries so they don't need to be packaged with every Lambda function.

In this project, the required library is:

* `snowflake-connector-python`

---

# Why do we need a Lambda Layer?

AWS Lambda only includes built-in Python libraries.

If our project uses external packages such as:

* snowflake-connector-python
* cryptography
* cffi
* requests

then we need to either:

1. Include them inside the deployment package, or
2. Create a Lambda Layer (Recommended)

Using a Layer keeps the Lambda deployment package small and allows multiple Lambda functions to reuse the same dependencies.

---

# Project Structure

Create a folder:

```text
snowflake-layer/
└── python/
```

The **python** folder is mandatory because AWS Lambda automatically loads packages from this directory.

---

# Step 1 – Create Folder

```bash
mkdir snowflake-layer
cd snowflake-layer
mkdir python
```

---

# Step 2 – Install Snowflake Connector

Install the package inside the **python** directory.

```bash
pip install snowflake-connector-python -t python/
```

If additional packages are required:

```bash
pip install requests -t python/
pip install boto3 -t python/
```

> Note:
> boto3 is already available in AWS Lambda by default, so installing it is usually unnecessary.

---

# Step 3 – Verify Folder Structure

The folder should look similar to:

```text
snowflake-layer/
└── python/
    ├── snowflake/
    ├── cryptography/
    ├── cffi/
    ├── _cffi_backend.cpython-311-x86_64-linux-gnu.so
    ├── typing_extensions.py
    └── ...
```

---

# Step 4 – Create ZIP File

From inside the **snowflake-layer** directory:

```bash
zip -r snowflake-layer.zip python
```

The ZIP file should contain:

```text
snowflake-layer.zip
└── python/
```

**Do NOT zip the contents directly.**

Correct:

```text
snowflake-layer.zip
└── python/
```

Incorrect:

```text
snowflake-layer.zip
├── snowflake/
├── cryptography/
└── cffi/
```

---

# Step 5 – Upload Layer to AWS

AWS Console

Lambda → Layers → Create Layer

Upload:

```
snowflake-layer.zip
```

Choose:

* Runtime: Python 3.11
* Architecture: x86_64 (or whatever your Lambda uses)

---

# Step 6 – Attach Layer

Open your Lambda Function.

Go to:

```
Layers
↓
Add a Layer
↓
Choose Custom Layer
↓
Select Version
↓
Add
```

---

# Runtime Compatibility

The Lambda Runtime and Layer Runtime **must match**.

Example:

| Lambda Runtime | Layer         |
| -------------- | ------------- |
| Python 3.11    | ✅ Python 3.11 |
| Python 3.12    | ❌ Python 3.11 |
| Python 3.14    | ❌ Python 3.11 |

If they don't match, errors such as the following may occur:

```text
No module named '_cffi_backend'
```

---

# Common Errors

## 1. No module named '_cffi_backend'

Cause:

* Layer built for a different Python version.
* Layer built on an incompatible operating system.
* Missing compiled dependencies.

Solution:

* Use the correct runtime.
* Rebuild the layer using Amazon Linux or Docker.

---

## 2. Unable to import module

Cause:

* Layer not attached.
* Wrong ZIP structure.

Solution:

Make sure the ZIP contains:

```text
python/
```

---

## 3. Invalid ZIP Structure

Wrong:

```text
snowflake-layer.zip
├── snowflake
├── cryptography
```

Correct:

```text
snowflake-layer.zip
└── python/
    ├── snowflake
    ├── cryptography
```

---

# Best Practice

Instead of creating the Layer on Windows, use Docker or Amazon Linux because compiled libraries (like cryptography and cffi) are fully compatible with AWS Lambda.

Example:

```bash
docker run -it amazonlinux:2023 bash
```

Inside the container:

```bash
yum install -y python3-pip zip

mkdir python

pip3 install snowflake-connector-python -t python

zip -r snowflake-layer.zip python
```

---

# Lessons Learned

While working on this project, I encountered the following issues:

* Python runtime mismatch (3.14 vs 3.11)
* Missing `_cffi_backend`
* Invalid Secrets Manager JSON
* Incorrect S3 bucket name (extra space)
* Lambda Layer compatibility

After fixing these issues, the Lambda function successfully:

* Retrieved exchange rates from the API
* Stored the JSON file in Amazon S3
* Loaded the data into Snowflake using a Stored Procedure

---

# References

AWS Lambda Layers

https://docs.aws.amazon.com/lambda/latest/dg/chapter-layers.html

Snowflake Python Connector

https://docs.snowflake.com/en/developer-guide/python-connector/python-connector
