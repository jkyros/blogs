# Important Deliveries to Paused Pools

## The MCO Added A New Critical Alert

Along with OpenShift 4.11 the Machine Config Operator (MCO) added a new feature -- [an alert](https://github.com/openshift/runbooks/blob/master/alerts/machine-config-operator/MachineConfigControllerPausedPoolKubeletCA.md) to notify users when a certificate rotation is being prevented by a paused MachineConfig pool.

For users who aren't using the Machine Config Pool's `Pause` feature, you will never experience this alert.

For users who *are* using the `Pause` feature, especially those who leave their machine config pools paused long-term due to maintenance windows, etc, this will hopefully make the risks and tradeoffs more apparent, and protect you from weird cluster failure modes.

## But Why Did We Do That?

The short answer is: Because we are trying to help you

The long answer is something like: The consequences of long-term pausing a `MachineConfigPool`are not immediately clear and predictable without a detailed understanding of internal OpenShift workings, and some of those consequences are very unfortunate. 

We also don't really know what you're doing all the time or why you're doing it, so sometimes it's hard for us to know how to help without getting in the way.

## The Relevant Parts of the MCO

If you aren't familiar with entirely how the MCO works, you can check out [the documentation](https://github.com/openshift/machine-config-operator/tree/master/docs). several

This is an oversimplification, but for our purposes here, in "standard operation" the MCO's `machine-config-operator`:

1. Watches and reacts when certain cluster objects change
2. Regenerates a CRD object called `ControllerConfig` when those objects change

And then the MCO's `machine-config-controller`:

4. Watches for `ControllerConfig` changes
5. Uses `ControllerConfig` to generate/regenerate template `MachineConfig` and other special controller-generated config
6. Watches for `MachineConfig` changes 
7. Merges alllllll that `MachineConfig` (template + user + generated) together deterministically into `rendered-configs` grouped by `MachineConfigPool`
8. Coordinates assignment of that rendered-config to nodes via a `desiredConfig` annotation (and the resulting reboots and drains) across a pool

And *then* the `machine-config-daemon` on the `Node`:

9. Applies the specified `desiredConfig` to its `Node`

So objects change, the MCO reacts to it, sqishes it all of the config into a `rendered-config`, and then assigns that `rendered-config` to `Nodes` `Pool.Spec.MaxUnavailable` at a time, where the `Node's` `machine-config-daemon` reacts to it and applies the `desiredConfig` config.  

(There are a bunch of bootstrapping/special cases, but pause does not currently affect those, so I'm going to hand-wave those away for our purposes here)

## ...Pause?

Some of you have never used `Pause` and may have no idea what I'm talking about, so before I go too far:

* The `MachineConfigPool` has a field in its CRD named`.Spec.Pause`. It has actually been in there [since the "beginning"](https://github.com/abhinavdahiya/machine-config-operator/blob/1617f8012dbb72db8684abd7351b6c3b0c7ccabf/pkg/apis/machineconfiguration.openshift.io/v1/types.go#L184-L186) of the MCO when the machineconfig types were [initially added](https://github.com/openshift/machine-config-operator/pull/2).
* When set to `true` that field instructs the MCO's node controller uses it to prevent configuration changes from being rolled out to nodes.

It's useful when you want to prevent the MCO from changing node config, or when you want to stack up a whole bunch of changes and only take a single reboot.

Much like the initial `MachineConfigPool` field, the [logic around pause](https://github.com/abhinavdahiya/machine-config-operator/blob/7357cc0318ab2484cc5dd4925956badd340d3bd6/pkg/controller/node/controller.go#L389) was part of the original `machine-config-controller` [design](https://github.com/openshift/machine-config-operator/pull/2) also.

So the sagely original developers knew that there would be cases where you wouldn't want configuration to roll out instantly, and they planned for it. What they maybe didn't plan for (and couldn't really have planned for) was all of the users who might eventually use it, the reasons it might be eventually used, and the long-term consequences and interactions resulting from its use.

## So What Does Pause Actually Do?

The `.Spec.Pause` field in the `MachineConfigPool` CRD only does one very specific thing: *It prevents the MCO's node_controller from changing a node's desiredConfig annotation.*

That's it. That's all it does. It happens [right here](https://github.com/openshift/machine-config-operator/blob/cad20f7101719838aaa87c066430987303500fa6/pkg/controller/node/node_controller.go#L777) and we just return early during the pool sync without doing anything else.

Everything else in the MCO happens as normal. It doesn't pause the entire MCO. It's doesn't even pause "most" of the MCO, it just keeps that one thing from happening -- which conveniently is enough to prevent any new MCO config changes from being applied to any nodes.

(NOTE: Emphasis on *new* config changes here because if `desiredConfig` has already been set on a node, and the `machine-config-daemon`is already in the process of applying a configuration, pause *will not stop it*, because that's not what it stops).

To reiterate -- things pause stops:

* The MCO's `node_controller` from setting the `desiredConfig` annotation on nodes in a pool

This, by extension, does stop the `machine-config-daemon` from applying configuration to nodes in the paused pool because:

* The `machine-config-daemon` only applies config when the `desiredConfig` annotation changes

Things `Pause` does *not* stop:

* New `ControllerConfig` generation
* New template `MachineConfig` generation/regeneration
* New rendered `MachineConfig` generation
* A user manually setting `desiredConfig` for a node (please do not do this)
* A toolchest falling down the stairs

## The MCO As The Postal Service

So in general MCO really likes to think of itself like the postal service. One of its most common defenses of whether or not we should be inspecting or surfacing some sort of config-specific data is **"we don't look in the envelope, we just deliver it."**

Yes, there are places where maybe we [cheat a little](https://github.com/openshift/machine-config-operator/blob/a2f16fc7fc57f224d06f6ca4c4fb6104b1a3d7c5/pkg/operator/sync.go#L246) and snarf an object out of the cluster and [pack](https://github.com/openshift/machine-config-operator/blob/a2f16fc7fc57f224d06f6ca4c4fb6104b1a3d7c5/pkg/operator/sync.go#L939) the [envelope](https://github.com/openshift/machine-config-operator/blob/a2f16fc7fc57f224d06f6ca4c4fb6104b1a3d7c5/templates/common/_base/files/kubelet-ca.yaml#L5) (which is maybe why it's okay for us to [take a peek](https://github.com/openshift/machine-config-operator/blob/cad20f7101719838aaa87c066430987303500fa6/pkg/controller/node/node_controller.go#L1203) in that envelope later), but for the most part we try to hold to that philosophy.

So you have this big sprawling MCO postal service:

* it services every city (each `MachineConfigPool`)
* it has a bunch of intake mailboxes that people drop mail in (templates, cluster object changes, `MachineConfig)`
* it has sorting and consolidation offices (`render_controller`)
* it has the "last mile" letter carrier (`node_controller`)
  * that puts the actual mail (`rendered MachineConfig`)
  * in your mailbox (`Node` `desiredConfig` annotation)
  * so that the recipient (`machine-config-daemon`) can open it and deal with it.

This metaphorical postal service also has a very weird feature (`Pause`), wherein:

* the Mayor  (`Administrator`) of a city (`MachineConfigPool`) can call in and say "stop delivering mail to my city until I tell you that you can resume delivery" (`.Spec.Pause`) and the postal service will obey and stop that "last mile" letter carrier (`node_controller`) for that city from delivering mail.

I know, it's not 100% perfect, but it should get us where we need to go

<img src=./PauseDiagramOversimplified.svg style="width: 50%; height: 50%"/>
(this diagram is obviously a hand-wavy oversimplification)

## The Mayor Said To Stop, So We Did?

So let's say the mayor of the city of `worker` has called the postal service and said to stop delivering mail.

As we said, this `Pause` in mail delivery doesn't stop anything other than that last mile letter carrier delivering the mail to residents' mailboxes. People still mail letters, the mail still gets picked up, and packed and sorted, and loaded onto mail vehicles, but the mail *never gets delivered*.

Residents, of `worker` of course, could continue to open and deal any mail that had previously been delivered to their mailboxes, but they wouldn't get any new mail (and they obviously can't open and use mail they don't have).

To those doing the mailing, everything appears to be functioning perfectly -- it all still gets collected.

But to those expecting to receive mail, things are different: nothing shows up. Maybe that's okay for the most part. Maybe there actually wasn't any mail set to be delivered. Maybe the residents are busy and can't or don't want to deal with their mail right now anyway.  

But let's say there is a resident of `worker` out there waiting for a really important piece of mail. Something with a deadline. Something like...their new passport, so they can go on vacation in a week. If they don't get it in time, they can't travel, and if they can't travel they're going to miss their trip.  

Well, now we have problems because this particular mail is very important, and it's also not being delivered because the Mayor told the postal service not to.

## Your Nodes Were Expecting A Very Important Letter

This "important mail not being delivered" scenario is exactly why we had to add the alerting around `Pause`. In our case, the "important mail" is the certificate bundle that controls trust between the `Node` and the cluster -- it is "mailed" by the apiserver operator, and it expects it to be delivered in a timely fashion. If the old bundle expires before the new one shows up, your `Node` stops talking to the cluster and becomes unmanageable.

More specifically, I'm referring to the  `kube-apiserver-to-kubelet-signer` certificate, which is [managed by the kube-apiserver operator](https://github.com/openshift/cluster-kube-apiserver-operator/blob/13c249ee91319da328a169a4e604ed863e049ff1/pkg/operator/certrotationcontroller/certrotationcontroller.go#L176), which it [rotates every 292 days](https://github.com/openshift/cluster-kube-apiserver-operator/blob/13c249ee91319da328a169a4e604ed863e049ff1/vendor/github.com/openshift/library-go/pkg/operator/certrotation/signer.go#L20) (lifetime is 365 days, rotates at 80%).

It rotates it by updating the `kube-apiserver-to-kubelet-signer` secret object in the `openshift-kube-apiserver-operator` namespace.

And I said earlier, the MCO pays attention to many cluster objects so it can react when they change and reconcile. That apiserver secret just so happens to be one of the cluster objects that the `machine-config-operator` pod [cares about](https://github.com/openshift/machine-config-operator/blob/b85ed4ebe6c0b61b31db51a63802407bc66c9f7c/pkg/operator/sync.go#L260).

So when the apiserver rotates that certificate and the secret object changes, the `machine-config-operator` notices and regenerates clusterconfig/renderconfig, which results in a new set of rendered `MachineConfig` being generated containing that new certificate bundle that will totally save the day!

And that super important rendered `MachineConfig` just...sits there behind `Pause`. :(

## We're Telling the Mayor There Will Be Trouble

So that's what the alert is for. Back in our metaphor, the alert we added is probably something like: "when we know 'important' mail is waiting at the post office, make sure the Mayor knows, call him on the Big Red Phone, and he can decide what to do".

## But If It's So Important, Why Not Just Deliver It?

So now you're probably thinking "well why not just let that "important mail" -- that certificate -- through?". It is "platform" config, not user config. We know where it's going and can sneak it in without a reboot, why not just do that?

Well, the short version is that:

* It's sets a weird precedent that pause isn't really pause
* We don't want to lie to you

The "contract" right now that we have with users is more or less that when pools are paused, no configuration gets rolled out. It's been that way since *the beginning of time* and people are used to it. **Pause doesn't lie**. The MCO will not move a node to a new configuration if the pool is paused.

## Lie To Me

Alright, so what if we did decide to lie to you? We tell you your pool is paused, but we give a special dispensation to "important config" (however we classify it) and we just sneak it though and have the `machine-config-daemon` just write it to the node.

That means (among other things):

1. We have to have a way to know what is/isn't "important" enough to bypass pause
2. We have to tell you about that special case (maybe in a "red box" in our documentation that you may or may not happen to read)
3. We now have to apply only a subset of config (which the `machine-config-daemon` does not currently do)
4. Our `currentConfig` annotation will be a lie, because we aren't actually in that `currentConfig` anymore (`MachineConfig` doesn't really have a status that supports "nuance" at this point -- either you're in a config and that's good, or your not and that's a problem)
5. Were there to be a severe enough error while sneaking through that "important" config, it could have stability consequences that the user didn't get to plan for.

## We'd Rather Tell the Truth

So there isn't isn't an immediately obvious solution here without having to trade something:

* Maybe we break out certificates into another operator that only reconciles certificates (but now the node config is a combination of  "whatever is in machineconfig" + "whatever the 'certificate operator' applied")
* Maybe we do a "one-off" rendered-config that's special that only contains the certificate changes (but now we're moving off of the current config "without the user's consent")
* Or maybe the alert is enough (but the user has to fix the issue themselves and potentially take an unwanted outage depending on what kind of config is pending)

One of those trades might eventually end up being right, but we're not quite sure yet, and regardless the ramifications would need to be fully communicated and understood.

## How Do You Want Your Mail Delivered?

So like I said at the beginning -- we're not always entirely sure how you're going to use what we put out there. We want to help you, but we also want to stay out of your way. Ideally we would be 100% helpful and 0% in your way, but understandably that's difficult given the wide variety of use cases.

 Maybe you'll use `Pause` how we thought it would be used. Maybe you will use it an entirely different way, but either way [we would love to know what you think](https://github.com/openshift/machine-config-operator/issues).

When the MCO isn't worrying about the consequences of `Pause`, we're working on cool new things like [OCP CoreOS layering](https://github.com/openshift/enhancements/blob/master/enhancements/ocp-coreos-layering/ocp-coreos-layering.md) -- check it out!
