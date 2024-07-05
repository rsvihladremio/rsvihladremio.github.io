+++
title = "Helm --set Will Surprise You"
date = 2024-07-05
+++

## History

I have a long tortured history with Helm. I was at a tiny startup many years ago that only wanted to use the latest and greatest for our core product (this was a bad idea btw). I was tasked with implementing a feature using the very bleeding edge [Open Service Broker API](https://www.openservicebrokerapi.org/) (this was 2016 mind you), and adding support for custom Helm charts to be uploaded to the catalog. This got nowhere quickly, as just getting things to compile back then was complicated.

I being the sort of "get it done" type of developer who sometimes will do horrible things to ship, I jettisoned Open Service Broker API and settled on the advanced pattern of "just using Helm" so I wrote a service that just shelled out to Helm:

* The customer would be presented with a list of charts to pick from, as well as they were given an upload box for their custom helm charts.
* I would populate a form from the options and values in the values.yaml
* The customer could then pass whatever configuration options they wanted and click deploy
* On the backend i'd read the options passed and use --set with the option key and custom value to pass them onto Helm. Effectively I called `helm install my-deployment --set myoption=1 ./chartdir. Suprisingly to everyone, myself included, this worked _very_ _well_.

This experienced turned out to be useful, the startup died under the weight of of hot bleeding edge tech and an MVP with endless feature lists, but the skills I learned with Go, Kubernetes, and Helm paid my bills in the years to come. I ended up being "an expert" in these technologies mostly by virtue of having done a bunch of stupid things early on in the hype cycle.

### Bad Habits

With my success using `--set` I got very much in the habit of just using values.yaml as a defaults file and then overriding it with `--set` for all deployments. I would even script this or automate this as much as possible. 

However, at some point I wanted to get away from people running custom scripts to use an industry standard tool, about a month ago I removed the scripts and just started documenting how to run Helm in my projects. I was proud to follow industry practice. But then I got lots of the following after upgrades:

```bash
kubectl get pods -n myns
NAME                          READY   STATUS             RESTARTS   AGE
my-deployment-66b97b646f-bxg8f   0/1     ImagePullBackOff   0          83s
my-deployment-bcd586b45-mpnpd    1/1     Running            0          42h
```

When I'd go to look at the deployment, I would see an old image name (that was no longer valid) or some other stale configuration.

```bash
 kubectl get deployment -o yaml  | grep image:
        - image: my-deployment:v0.20.4

```

I would often ignore this and fixed the deployment manually, but it would return every deploy. Finally, I dug a bunch through GitHub and found the [following issue](https://github.com/helm/helm/issues/12687), with this helpful text

    I found the issue. It was previously supplied individual value overrides being provided by --set getting persisted in the release secrets and then re-applied when the manifest got rendered again. Using helm -n sentry get values sentry revealed the saved override and then using --reset-values when running helm upgrade or helm diff upgrade removed its application

So I looked at my own configuration and sure enough I had the values from the last time I used --set to override the yaml.

```bash
helm -n myns get values my-deployment
USER-SUPPLIED VALUES:
image:
  tag: 0.20.11
replicaCount: 3
```

I fixed it with the following command:

```bash
helm upgrade -n myns my-deployment ./my-chart --reset-values
```

Now when I retrieved the values I got a nice blank and subsequent upgrades worked as expected.

```bash
helm -n myns get values my-deployment       
USER-SUPPLIED VALUES:
null
```

## Takeway

I learned three important things from this:

* Avoid --set (or be prepared to always use the same options forever)
* Use the values.yaml to set your values
* If you ever have weird values showing up in a helm upgrade get it with `helm get values`