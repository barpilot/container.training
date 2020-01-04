# Daemon sets

- We want to scale `rng` in a way that is different from how we scaled `worker`

- We want one (and exactly one) instance of `rng` per node

- We *do not want* two instances of `rng` on the same node

- We will do that with a *daemon set*

---

## Why not a deployment?

- Can't we just do `kubectl scale deployment rng --replicas=...`?

--

- Nothing guarantees that the `rng` containers will be distributed evenly

- If we add nodes later, they will not automatically run a copy of `rng`

- If we remove (or reboot) a node, one `rng` container will restart elsewhere

  (and we will end up with two instances `rng` on the same node)

- By contrast, a daemon set will start one pod per node and keep it that way

  (as nodes are added or removed)

---

## Daemon sets in practice

- Daemon sets are great for cluster-wide, per-node processes:

  - `kube-proxy`

  - `weave` (our overlay network)

  - monitoring agents

  - hardware management tools (e.g. SCSI/FC HBA agents)

  - etc.

- They can also be restricted to run [only on some nodes](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/#running-pods-on-only-some-nodes)

---

## Creating a daemon set

<!-- ##VERSION## -->

- Unfortunately, as of Kubernetes 1.15, the CLI cannot create daemon sets

--

- More precisely: it doesn't have a subcommand to create a daemon set

--

- But any kind of resource can always be created by providing a YAML description:
  ```bash
  kubectl apply -f foo.yaml
  ```

--

- How do we create the YAML file for our daemon set?

--

  - option 1: [read the docs](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/#create-a-daemonset)

--

  - option 2: `vi` our way out of it

---

## Creating the YAML file for our daemon set

- Let's start with the YAML file for the current `rng` resource

.exercise[

- Dump the `rng` resource in YAML:
  ```bash
  kubectl get deploy/rng -o yaml >rng.yml
  ```

- Edit `rng.yml`

]

---

## "Casting" a resource to another

- What if we just changed the `kind` field?

  (It can't be that easy, right?)

.exercise[

- Change `kind: Deployment` to `kind: DaemonSet`

<!--
```bash vim rng.yml```
```wait kind: Deployment```
```keys /Deployment```
```keys ^J```
```keys cwDaemonSet```
```keys ^[``` ]
```keys :wq```
```keys ^J```
-->

- Save, quit

- Try to create our new resource:
  ```
  kubectl apply -f rng.yml
  ```

]

--

We all knew this couldn't be that easy, right!

---

## Understanding the problem

- The core of the error is:
  ```
  error validating data:
  [ValidationError(DaemonSet.spec):
  unknown field "replicas" in io.k8s.api.extensions.v1beta1.DaemonSetSpec,
  ...
  ```

--

- *Obviously,* it doesn't make sense to specify a number of replicas for a daemon set

--

- Workaround: fix the YAML

  - remove the `replicas` field
  - remove the `strategy` field (which defines the rollout mechanism for a deployment)
  - remove the `progressDeadlineSeconds` field (also used by the rollout mechanism)
  - remove the `status: {}` line at the end

--

- Or, we could also ...

---

## Use the `--force`, Luke

- We could also tell Kubernetes to ignore these errors and try anyway

- The `--force` flag's actual name is `--validate=false`

.exercise[

- Try to load our YAML file and ignore errors:
  ```bash
  kubectl apply -f rng.yml --validate=false
  ```

]

--

🎩✨🐇

--

Wait ... Now, can it be *that* easy?

---

## Checking what we've done

- Did we transform our `deployment` into a `daemonset`?

.exercise[

- Look at the resources that we have now:
  ```bash
  kubectl get all
  ```

]

--

We have two resources called `rng`:

- the *deployment* that was existing before

- the *daemon set* that we just created

We also have one too many pods.
<br/>
(The pod corresponding to the *deployment* still exists.)

---

## `deploy/rng` and `ds/rng`

- You can have different resource types with the same name

  (i.e. a *deployment* and a *daemon set* both named `rng`)

- We still have the old `rng` *deployment*

  ```
NAME                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/rng        1         1         1            1           18m
  ```

- But now we have the new `rng` *daemon set* as well

  ```
NAME                DESIRED  CURRENT  READY  UP-TO-DATE  AVAILABLE  NODE SELECTOR  AGE
daemonset.apps/rng  2        2        2      2           2          <none>         9s
  ```

---

## Too many pods

- If we check with `kubectl get pods`, we see:

  - *one pod* for the deployment (named `rng-xxxxxxxxxx-yyyyy`)

  - *one pod per node* for the daemon set (named `rng-zzzzz`)

  ```
  NAME                        READY     STATUS    RESTARTS   AGE
  rng-54f57d4d49-7pt82        1/1       Running   0          11m
  rng-b85tm                   1/1       Running   0          25s
  rng-hfbrr                   1/1       Running   0          25s
  [...]
  ```

--

The daemon set created one pod per node, except on the master node.

The master node has [taints](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) preventing pods from running there.

(To schedule a pod on this node anyway, the pod will require appropriate [tolerations](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/).)

.footnote[(Off by one? We don't run these pods on the node hosting the control plane.)]

---

## Is this working?

- Look at the web UI

--

- The graph should now go above 10 hashes per second!

--

- It looks like the newly created pods are serving traffic correctly

- How and why did this happen?

  (We didn't do anything special to add them to the `rng` service load balancer!)
