---
title: "AWS Status Portal, Terraform & Ansible"
date: 2021-07-13T13:43:11+01:00
draft: false
authors:
  - rastamouse
---



[Skip to the bottom of the page for the video demo]

<br/>

In a [previous](/posts/aws-status-portal/) blog post, I outlined how to build a status portal for EC2 instances running in AWS.  The portal itself was a Blazor app which used the AWS SDK to interact with (list/start/stop) the virtual machines.

[Heath Adams](https://twitter.com/thecybermentor/status/1413573964920758274) posted a question to Twitter about building a CMS that could use AWS and Terraform to manage lab deployments.

Obviously, I already had code that could interact with existing VMs in AWS, but it got me wondering how difficult it would be to integrate Terraform into the mix.  I had also already published an article on [rastamouse.me](https://rastamouse.me/infrastructure-as-code-terraform-ansible/) about deploying and managing VMs in AWS using Terraform and Ansible; so this could just be a case of combining the two...?  🤞

However, one thing I wanted to avoid was managing files locally on the webserver running my portal - that includes the Terraform configuration and state files; and the Ansible playbooks.  I wanted a solution that was more API-driven, so that the actual web development portion was as simple as possible.

To handle the Terraform side of things, I decided to check out [Terraform Cloud](https://www.terraform.io/cloud).

<br/>



## Terraform Cloud

Terraform Cloud allows you to create workspaces which contain your Terraform configurations, shared variable values, current and historical Terraform state, and run logs.  Once a workspace has been created and configurations uploaded, runs can be triggered manually or via the API.

When a run is triggered, Terraform goes through a workflow of "Planning" -> "Planned" -> Confirmation Required" -> "Applying" -> "Applied".  You must explicitly confirm a plan before it can be applied, and that can only be done when it's in a "planned" state.

The Terraform API is documented [here](https://www.terraform.io/docs/cloud/api/index.html) which is a useful reference, but I did use the [Tfe.NetClient](https://github.com/everis-technology/Tfe.NetClient) package in my application to simplify the API calls.

Terraform Cloud can also trigger webhooks when a run's state changes, which is brilliant for updating the portal's UI.  I implemented a controller to handle the incoming hook and trigger an event in the same way as in the previous post.

<br/>



## Ansible

If you read my Terraform + Ansible post, you'll see that I used Terraform's `local-exec` provisioner to execute `ansible-playbook` against the newly provisioned VMs.  However, (as far as I can tell) the containers used by Terraform Cloud don't have this available, so I needed a different method of triggering Ansible after the VMs have been deployed.

The solution I came up with was to deploy a dedicated Ansible control node in AWS.  The idea being that this would run constantly and wait to be triggered when a new lab is deployed.  I also found that Ansible has several plugins for dynamic inventory, including one for AWS which allows it to query your account and get back a list of EC2 instances.

Ultimately, this allowed me to execute playbooks against instances that have particular tags.

```yaml
- become: no
  hosts: tag_Name_Kali
  name: kali-setup
  tasks:
    - name: Hushlogin
      file:
        path: /home/kali/.hushlogin
        state: touch
        mode: u=rw,g=r,o=r
```

```bash
$ ansible-playbook -u kali --private-key deployment.pem playbooks/kali-setup.yaml
```



<br/>

Furthermore, you can check that the host is available and ready before trying to execute the playbook with something like:

```bash
$ ansible tag_Name_Kali -m ping -u kali --private-key deployment.pem
```



<br/>

To interact with Ansible from my web portal, I was very lazy and just used an SSH client library.  I admittedly don't know Ansible that well, so I'm sure there's a better way.



## Portal

I created a new interface for Terraform to dependency inject in my page.



```c#
public interface ITerraform
{
    Task<Attributes> GetWorkspaceAttributes(string workspaceId);
    Task DeployLab(string workspaceId);
    Task DestroyLab(string workspaceId);
}
```



<br/>

The implementation is not that complicated thanks to the `TfeClient`.

```c#
public class Terraform : ITerraform
{
    private readonly TfeClient _tfeClient;

    public Terraform(string bearer)
    {
        var client = new HttpClient();
        var config = new TfeConfig(bearer, client);
            
        _tfeClient = new TfeClient(config);
    }

    public async Task<Attributes> GetWorkspaceAttributes(string id)
    {
        var response = await _tfeClient.Workspace.ShowAsync(id);
        return response.Data.Attributes;
    }

    public async Task DeployLab(string id)
    {
        var request = new RunsRequest();
        request.Data.Attributes.Message = "Triggered via API";
        request.Data.Relationships.Workspace.Data.Id = id;
        request.Data.Relationships.Workspace.Data.Type = "workspaces";

        var response = await _tfeClient.Run.CreateAsync(request);
        await ApplyRun(response.Data.Id);
    }

    public async Task DestroyLab(string id)
    {
        var request = new RunsRequest();
        request.Data.Attributes.Message = "Triggered via API";
        request.Data.Attributes.IsDestroy = true;
        request.Data.Relationships.Workspace.Data.Id = id;
        request.Data.Relationships.Workspace.Data.Type = "workspaces";

        var response = await _tfeClient.Run.CreateAsync(request);
        await ApplyRun(response.Data.Id);
    }

    private async Task ApplyRun(string id)
    {
        var ready = false;
            
        while (!ready)
        {
            await Task.Delay(1000);
            var run = await _tfeClient.Run.ShowAsync(id);
            ready = run.Data.Attributes.Status.Equals("planned", StringComparison.OrdinalIgnoreCase);
        }

        await _tfeClient.Run.ApplyAsync(id, null);
    }
}
```



<br/>

On my main web page, I can then call `await _terraform.GetWorkspaceAttributes(_labId);` in the `OnInitializedAsync()` method and populate my HTML with the values.  I also hardcoded the lab ID on the page, because PoC, although you can list all of your workspaces and just iterate through each one.

```c#
<div class="card" style="margin: 20px; width: 450px">
	@if (_labAttributes is not null)
	{
    	<div class="card-header">@_labAttributes.Name</div>
	    <div class="card-body">
	    <p class="card-title">@_labAttributes.Description</p>
    	<p>@_runStatus</p>

	    <button class="btn btn-primary" disabled="@_deployDisabled" @onclick="@(async () => await DeployLab())">Deploy</button>
	    <button class="btn btn-secondary" disabled="@_destroyDisabled" @onclick="@(async () => await DestroyLab())">Destroy</button>
    	</div>
	}
</div>
```



<br/>

Wiring the buttons up is easy with the interface.

```c#
private async Task DeployLab()
{
    _deployDisabled = true;
    await _terraform.DeployLab(_labId);
}

private async Task DestroyLab()
{
    _destroyDisabled = true;
    await _terraform.DestroyLab(_labId);
}
```



<br/>

I also made some improvements to the UI by putting disable checks on the buttons.  Here are two example for starting and stopping instances.  An instance can only be started if it's in a "stopped" state, and can only be stopped if it's in a "started" state.

```c#
private bool CanStopInstance(string instanceId)
{
    var instance = _instances.FirstOrDefault(i => i.InstanceId.Equals(instanceId));
    var condition = instance?.State.Name == InstanceStateName.Running;
    return !condition;
}
    
private bool CanStartInstance(string instanceId)
{
    var instance = _instances.FirstOrDefault(i => i.InstanceId.Equals(instanceId));
    var condition = instance?.State.Name == InstanceStateName.Stopped;
    return !condition;
}
```



<br/>



It was also fun to throw in an Apache Guacamole server and provide a console button to interact directly with the VM.

Here's the full thing in action (watch in x2 speed for sanity):

<iframe width="1214" height="692" src="https://www.youtube.com/embed/trfq83RKz9A" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
