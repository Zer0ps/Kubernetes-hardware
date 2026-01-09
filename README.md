# Kubernetes-hardware

A guide to **manually installing Kubernetes (The Hard Way)** for those who want to **understand Kubernetes from the inside**.

No `kubeadm`, no cloud magic, no hidden abstractions.
Every Kubernetes component is installed and configured **by hand** - from TLS and etcd to kubelet and pod networking.

---

## Why

Most guides teach you how to **use** Kubernetes, not how to **understand** it.
This repository focuses on architecture, dependencies, and the real mechanisms behind a working cluster.

If you want to know:

* what actually runs
* how components communicate
* why the cluster works (or breaks)

---

## What’s inside

* Full manual Kubernetes installation
* Jumpbox, control plane, and worker nodes
* TLS, kubeconfigs, RBAC
* etcd, kube-apiserver, scheduler, controller-manager
* containerd, kubelet, kube-proxy
* Manual pod networking
* Smoke tests for a working cluster

---

## Who this is for

* aspiring DevOps / SRE engineers
* system administrators
* anyone tired of “just run `kubeadm init`”

Not for quick production setups.
For **real understanding**.

---

## Outcome

After completing this guide, you’ll understand Kubernetes as a system - not just a set of commands.

---

# ⚠️ Important ⚠️

This project is **for educational purposes only**.

It is **not intended for production use** and **must not be deployed in real environments**.
Use it **only locally**, in labs or test setups, to learn how Kubernetes works internally.

For production clusters, always use supported tools and best practices.

---
