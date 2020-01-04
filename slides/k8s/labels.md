# Labels and selectors

- The `rng` *service* is load balancing requests to a set of pods

- That set of pods is defined by the *selector* of the `rng` service

.exercise[

- Check the *selector* in the `rng` service definition:
  ```bash
  kubectl describe service rng
  ```

]

- The selector is `app=rng`

- It means "all the pods having the label `app=rng`"

  (They can have additional labels as well, that's OK!)

---

## Selector evaluation

- We can use selectors with many `kubectl` commands

- For instance, with `kubectl get`, `kubectl logs`, `kubectl delete` ... and more

.exercise[

- Get the list of pods matching selector `app=rng`:
  ```bash
  kubectl get pods -l app=rng
  kubectl get pods --selector app=rng
  ```

]

But ... why do these pods (in particular, the *new* ones) have this `app=rng` label?

---

## Where do labels come from?

- When we create a deployment with `kubectl create deployment rng`,
  <br/>this deployment gets the label `app=rng`

- The replica sets created by this deployment also get the label `app=rng`

- The pods created by these replica sets also get the label `app=rng`

- When we created the daemon set from the deployment, we re-used the same spec

- Therefore, the pods created by the daemon set get the same labels

.footnote[Note: when we use `kubectl run stuff`, the label is `run=stuff` instead.]

---

## Updating load balancer configuration

- We would like to remove a pod from the load balancer

- What would happen if we removed that pod, with `kubectl delete pod ...`?

--

  It would be re-created immediately (by the replica set or the daemon set)

--

- What would happen if we removed the `app=rng` label from that pod?

--

  It would *also* be re-created immediately

--

  Why?!?

---

## Selectors for replica sets and daemon sets

- The "mission" of a replica set is:

  "Make sure that there is the right number of pods matching this spec!"

- The "mission" of a daemon set is:

  "Make sure that there is a pod matching this spec on each node!"

--

- *In fact,* replica sets and daemon sets do not check pod specifications

- They merely have a *selector*, and they look for pods matching that selector

- Yes, we can fool them by manually creating pods with the "right" labels

- Bottom line: if we remove our `app=rng` label ...

 ... The pod "disappears" for its parent, which re-creates another pod to replace it

---

class: extra-details

## Isolation of replica sets and daemon sets

- Since both the `rng` daemon set and the `rng` replica set use `app=rng` ...

  ... Why don't they "find" each other's pods?

--

- *Replica sets* have a more specific selector, visible with `kubectl describe`

  (It looks like `app=rng,pod-template-hash=abcd1234`)

- *Daemon sets* also have a more specific selector, but it's invisible

  (It looks like `app=rng,controller-revision-hash=abcd1234`)

- As a result, each controller only "sees" the pods it manages

---

## Removing a pod from the load balancer

- Currently, the `rng` service is defined by the `app=rng` selector

- The only way to remove a pod is to remove or change the `app` label

- ... But that will cause another pod to be created instead!

- What's the solution?

--

- We need to change the selector of the `rng` service!

- Let's add another label to that selector (e.g. `enabled=yes`) 

---

## Complex selectors

- If a selector specifies multiple labels, they are understood as a logical *AND*

  (In other words: the pods must match all the labels)

- Kubernetes has support for advanced, set-based selectors

  (But these cannot be used with services, at least not yet!)

---

## The plan

1. Add the label `enabled=yes` to all our `rng` pods

2. Update the selector for the `rng` service to also include `enabled=yes`

3. Toggle traffic to a pod by manually adding/removing the `enabled` label

4. Profit!

*Note: if we swap steps 1 and 2, it will cause a short
service disruption, because there will be a period of time
during which the service selector won't match any pod.
During that time, requests to the service will time out.
By doing things in the order above, we guarantee that there won't
be any interruption.*

---

## Adding labels to pods

- We want to add the label `enabled=yes` to all pods that have `app=rng`

- We could edit each pod one by one with `kubectl edit` ...

- ... Or we could use `kubectl label` to label them all

- `kubectl label` can use selectors itself

.exercise[

- Add `enabled=yes` to all pods that have `app=rng`:
  ```bash
  kubectl label pods -l app=rng enabled=yes
  ```

]

---

## Updating the service selector

- We need to edit the service specification

- Reminder: in the service definition, we will see `app: rng` in two places

  - the label of the service itself (we don't need to touch that one)

  - the selector of the service (that's the one we want to change)

.exercise[

- Update the service to add `enabled: yes` to its selector:
  ```bash
  kubectl edit service rng
  ```

<!--
```wait Please edit the object below```
```keys /app: rng```
```keys ^J```
```keys noenabled: yes```
```keys ^[``` ]
```keys :wq```
```keys ^J```
-->

]

--

... And then we get *the weirdest error ever.* Why?

---

## When the YAML parser is being too smart

- YAML parsers try to help us:

  - `xyz` is the string `"xyz"`

  - `42` is the integer `42`

  - `yes` is the boolean value `true`

- If we want the string `"42"` or the string `"yes"`, we have to quote them

- So we have to use `enabled: "yes"`

.footnote[For a good laugh: if we had used "ja", "oui", "si" ... as the value, it would have worked!]

---

## Updating the service selector, take 2

.exercise[

- Update the service to add `enabled: "yes"` to its selector:
  ```bash
  kubectl edit service rng
  ```

<!--
```wait Please edit the object below```
```keys /app: rng```
```keys ^J```
```keys noenabled: "yes"```
```keys ^[``` ]
```keys :wq```
```keys ^J```
-->

]

This time it should work!

If we did everything correctly, the web UI shouldn't show any change.

---

## Updating labels

- We want to disable the pod that was created by the deployment

- All we have to do, is remove the `enabled` label from that pod

- To identify that pod, we can use its name

- ... Or rely on the fact that it's the only one with a `pod-template-hash` label

- Good to know:

  - `kubectl label ... foo=` doesn't remove a label (it sets it to an empty string)

  - to remove label `foo`, use `kubectl label ... foo-`

  - to change an existing label, we would need to add `--overwrite`

---

## Removing a pod from the load balancer

.exercise[

- In one window, check the logs of that pod:
  ```bash
  POD=$(kubectl get pod -l app=rng,pod-template-hash -o name)
  kubectl logs --tail 1 --follow $POD

  ```
  (We should see a steady stream of HTTP logs)

- In another window, remove the label from the pod:
  ```bash
  kubectl label pod -l app=rng,pod-template-hash enabled-
  ```
  (The stream of HTTP logs should stop immediately)

]

There might be a slight change in the web UI (since we removed a bit
of capacity from the `rng` service). If we remove more pods,
the effect should be more visible.

---

class: extra-details

## Updating the daemon set

- If we scale up our cluster by adding new nodes, the daemon set will create more pods

- These pods won't have the `enabled=yes` label

- If we want these pods to have that label, we need to edit the daemon set spec

- We can do that with e.g. `kubectl edit daemonset rng`

---

class: extra-details

## We've put resources in your resources

- Reminder: a daemon set is a resource that creates more resources!

- There is a difference between:

  - the label(s) of a resource (in the `metadata` block in the beginning)

  - the selector of a resource (in the `spec` block)

  - the label(s) of the resource(s) created by the first resource (in the `template` block)

- We would need to update the selector and the template

  (metadata labels are not mandatory)

- The template must match the selector

  (i.e. the resource will refuse to create resources that it will not select)

---

## Labels and debugging

- When a pod is misbehaving, we can delete it: another one will be recreated

- But we can also change its labels

- It will be removed from the load balancer (it won't receive traffic anymore)

- Another pod will be recreated immediately

- But the problematic pod is still here, and we can inspect and debug it

- We can even re-add it to the rotation if necessary

  (Very useful to troubleshoot intermittent and elusive bugs)

---

## Labels and advanced rollout control

- Conversely, we can add pods matching a service's selector

- These pods will then receive requests and serve traffic

- Examples:

  - one-shot pod with all debug flags enabled, to collect logs

  - pods created automatically, but added to rotation in a second step
    <br/>
    (by setting their label accordingly)

- This gives us building blocks for canary and blue/green deployments
