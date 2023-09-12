---
title: Design af LeakTest Service
date: 12.09.2023 10:00
categories: [Nolek, Andet]
tags: [nolek,leaktest,microservices,systemdokumentation,systemdesign]
---

Her kommer et overblik over mit design af LeakTest Service. Et kort oprids:
* **Gateway** er den service hvis API eksponeres som ingangspunkt til de andre services.
* **LeakTestController** er API'et for LeakTest Service.
* **LeakTestRepo** er en abstraction af interaktionen med den InfluxDB som er tilkoblet.
* **LeakTest** repræsenterer modellen.
* **Converter** er ansvarlig for mapping fra c# objekt til InfluxDB point og omvendt.
* **InfluxDB** den tilknyttede InfluxDB.

<img src="/assets/images/leak_test_service.png" alt="Image should have been here.">
