+++
date = '2025-05-08T23:05:44+08:00'
draft = true
title = 'Handsomeboy'
+++
**11**
22
33

    apiVersion: v1
    kind: PersistentVolume
    metadata:
    name: example-pv
    spec:
    capacity:
        storage: 10Gi
    accessModes:
        - ReadWriteOnce
    hostPath:
        path: "/mnt/data"
