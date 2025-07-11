+++
title = "Speeding Up Kubernetes Controller Integration Tests with Ginkgo Parallelism"
date = 2025-07-14T00:00:00+01:00
+++

## TL;DR

Kubernetes controller integration tests can be **painfully slow** when run sequentially, especially as your suite grows. In this post, you'll learn how to:

- Run integration tests in **parallel** using the [Ginkgo](https://onsi.github.io/ginkgo/) testing framework.
- Share cluster configuration between parallel test processes.
- Reduce test times significantly. In our case, the time dropped from **around 11 minutes to just 1 minute and 40 seconds**.

This guide uses real examples from the [Kuadrant Limitador Operator project](https://github.com/Kuadrant/limitador-operator/pull/134).

---

## Why This Matters

When testing Kubernetes controllers, integration tests are often run against a **real cluster**, like one provisioned by [`kind`](https://kind.sigs.k8s.io/). These tests validate your controller’s behavior by applying resources, watching for events, and verifying changes in cluster state — just like in production.

But running against a real cluster isn’t cheap: applying manifests, registering CRDs, and waiting for resources to reconcile takes time. Multiply that by dozens or hundreds of test cases, and things slow down quickly.

### Real-World Impact

In the Limitador Operator project, we run integration tests against a shared `kind` cluster. By enabling Ginkgo’s parallel mode and optimizing setup using `SynchronizedBeforeSuite`, our total test runtime dropped from **~11 minutes** to just **~1 minute and 40 seconds** — a **6.5× improvement**.

---

## Parallel Execution with Ginkgo

[Ginkgo](https://onsi.github.io/ginkgo/) is a BDD-style testing framework for Go that supports **parallel test execution** out of the box. Here’s how to get started.

### 1. Install Ginkgo and Gomega

```sh
go get -u github.com/onsi/ginkgo/v2/ginkgo
go get -u github.com/onsi/gomega/...
````

### 2. Update Your Test Suite

Enable Ginkgo in your test files:

```go
package controllers

import (
    "testing"

    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
)

func TestAPIs(t *testing.T) {
    RegisterFailHandler(Fail)
    RunSpecs(t, "Controller Suite")
}
```

Now write your specs as usual:

```go
var _ = Describe("Parallel Test Example", func() {
    It("runs test 1", func() {
        time.Sleep(5 * time.Second)
        Expect(true).To(BeTrue())
    })

    It("runs test 2", func() {
        time.Sleep(5 * time.Second)
        Expect(true).To(BeTrue())
    })

    It("runs test 3", func() {
        time.Sleep(5 * time.Second)
        Expect(true).To(BeTrue())
    })
})
```

### 3. Run Tests in Parallel

Just add [`-p`](https://onsi.github.io/ginkgo/#spec-parallelization):

```sh
ginkgo -p
```

This runs your specs in parallel across multiple processes, each isolated from the others.

---

## Advanced: Sharing Setup Between Parallel Nodes

When running Ginkgo tests in parallel, each node is a separate process. That means you can't just reuse in-memory Go variables for shared setup like clients or cluster configs.

Since our tests run against a **pre-existing `kind` cluster**, we configure Ginkgo nodes to use the same kubeconfig. This is done with `UseExistingCluster: ptr.To(true)` and a `SynchronizedBeforeSuite` that passes cluster connection details to each test node.

> While we use `envtest.Environment` for convenience, it’s configured to connect to a real cluster (like `kind`) using `UseExistingCluster: true`, rather than spinning up a local API server.

### Key Concepts

* The **first node** performs cluster setup and returns marshalled config.
* **All nodes** deserialize the config and create clients independently.

### Code Example

This setup is adapted from the Limitador Operator project.

```go
var k8sClient client.Client
var testEnv *envtest.Environment

type SharedConfig struct {
    Host            string           `json:"host"`
    TLSClientConfig TLSClientConfig `json:"tlsClientConfig"`
}

type TLSClientConfig struct {
    Insecure bool    `json:"insecure"`
    CertData []byte  `json:"certData,omitempty"`
    KeyData  []byte  `json:"keyData,omitempty"`
    CAData   []byte  `json:"caData,omitempty"`
}

var _ = SynchronizedBeforeSuite(func() []byte {
    testEnv = &envtest.Environment{
        CRDDirectoryPaths:     []string{filepath.Join("..", "config", "crd", "bases")},
        ErrorIfCRDPathMissing: true,
        UseExistingCluster:    ptr.To(true), // Connects to pre-existing cluster (e.g. kind)
    }

    cfg, err := testEnv.Start()
    Expect(err).NotTo(HaveOccurred())

    sharedCfg := SharedConfig{
        Host: cfg.Host,
        TLSClientConfig: TLSClientConfig{
            Insecure: cfg.TLSClientConfig.Insecure,
            CertData: cfg.TLSClientConfig.CertData,
            KeyData:  cfg.TLSClientConfig.KeyData,
            CAData:   cfg.TLSClientConfig.CAData,
        },
    }

    data, err := json.Marshal(sharedCfg)
    Expect(err).NotTo(HaveOccurred())
    return data
}, func(data []byte) {
    var sharedCfg SharedConfig
    Expect(json.Unmarshal(data, &sharedCfg)).To(Succeed())

    cfg := &rest.Config{
        Host: sharedCfg.Host,
        TLSClientConfig: rest.TLSClientConfig{
            Insecure: sharedCfg.TLSClientConfig.Insecure,
            CertData: sharedCfg.TLSClientConfig.CertData,
            KeyData:  sharedCfg.TLSClientConfig.KeyData,
            CAData:   sharedCfg.TLSClientConfig.CAData,
        },
    }

    scheme := runtime.NewScheme()
    Expect(kubescheme.AddToScheme(scheme)).To(Succeed())

    var err error
    k8sClient, err = client.New(cfg, client.Options{Scheme: scheme})
    Expect(err).NotTo(HaveOccurred())
})
```

Also add:

```go
var _ = SynchronizedAfterSuite(func() {}, func() {
    err := testEnv.Stop()
    Expect(err).ToNot(HaveOccurred())
})
```

And to make logs work:

```go
func TestMain(m *testing.M) {
    logf.SetLogger(zap.New(zap.WriteTo(GinkgoWriter), zap.UseDevMode(true)))
    os.Exit(m.Run())
}
```

---

## Tips and Caveats

* ✅ Use `UseExistingCluster: ptr.To(true)` to reuse a `kind` or dev cluster and skip test API server startup.
* ⚠️ Watch out for race conditions or tests that implicitly assume global state.
* ⚠️ Parallel execution increases load — avoid oversaturating resource-constrained clusters like `kind`.

---

## Final Thoughts

Parallelizing your Kubernetes integration tests is one of the best improvements you can make to your test suite. With Ginkgo’s support and a small amount of setup code, you can significantly reduce test time and increase developer productivity.

### Test Runtime Comparison

| Configuration         | Runtime            |
|-----------------------|--------------------|
| Sequential (Default)  | ~11 minutes        |
| Parallel with Ginkgo  | **~1 minute 40 seconds** |

Try it out and enjoy faster feedback loops and smoother CI workflows.

