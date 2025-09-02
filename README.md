# IN163 - Implementing Exactly Once In Order delivery in SAP Integration Suite
This default template for SAP Samples repositories includes files for README, LICENSE, and REUSE.toml. All repositories on github.com/SAP-samples will be created based on this template.

## Description

This repository contains the material for the SAP TechEd 2025 session called IN163 - Implementing Exactly Once In Order delivery in SAP Integration Suite.

## Overview

This session allows you to implement integration scenarios supporting Exactly Once In Order delivery to ensure data consistency when exchanging messages via SAP Integration Suite, that is messages are delivered without duplication and in their original sequence. In the session we will focus on two options:
- Using exclusive queues in Cloud Integration
- Via partitioned queues of SAP Integration Suite, advanced event mesh.

For an introduction into Exactly Once In Order support in Cloud Integration, check out this [blog post](https://community.sap.com/t5/integration-blog-posts/ensuring-exactly-once-in-order-quality-of-service-in-cloud-integration/ba-p/14180026).

For more details about the actual implementaiton options of Exactly Once In Order in Cloud Integration, refer to the [design guidelines](https://help.sap.com/docs/integration-suite/sap-integration-suite/quality-of-service-exactly-once-in-order).

## Access to SAP Integration Suite tenant

For running through the exercises, we provide the following SAP Integration Suite tenants. The instructors will let you know which of the tenants are actually used for the exercises.

- Option 1: [**Workshop tenant eu-02a**](https://workshop-eu-02a.integrationsuite-cpi033.cfapps.eu10-005.hana.ondemand.com/shell/home).
- Option 2: [**Workshop tenant eu-02b**](https://workshop-eu-02b.integrationsuite-cpi035.cfapps.eu20-001.hana.ondemand.com/shell/home).
- Use the user **userXX** with **XX** your ID and password provided by the instructors.

## Exercises

As a prerequisite for the exercises, you first need to run through the following preparation steps.
- [Exercise 0 - Prepare the Exercises](exercises/ex0/)

The exercises are divided into the following two parts:
- [Exercise 1 - EOIO via Exclusive Queues on Cloud Integration](exercises/ex1/)
- [Exercise 2 - EOIO via Partitioned Queues using AEM](exercises/ex2/)


## Contributing
Please read the [CONTRIBUTING.md](./CONTRIBUTING.md) to understand the contribution guidelines.

## Code of Conduct
Please read the [SAP Open Source Code of Conduct](https://github.com/SAP-samples/.github/blob/main/CODE_OF_CONDUCT.md).

## How to obtain support

Support for the content in this repository is available during the actual time of the online session for which this content has been designed. Otherwise, you may request support via the [Issues](../../issues) tab.

## License
Copyright (c) 2024 SAP SE or an SAP affiliate company. All rights reserved. This project is licensed under the Apache Software License, version 2.0 except as noted otherwise in the [LICENSE](LICENSES/Apache-2.0.txt) file.
