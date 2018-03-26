---
layout: post
title: Permissions over components inside AEM
---



This article is for AEM developers that tries to understand how permissions are done in AEM and how they can add a sort of restrictions over components. 

Let's begin with the basics : 

### <strong>AEM Permissions</strong>

AEM has a built-in  authentication / authorization framwork, where you can create users and groups and assign read and write permissions over <strong>pages and folders</strong> (You can go to http://localhost:4502/useradmin and see permissions tab) 

When you assign a read  access for example to a page, AEM is generating a permission node  called  <code>rep:policy</code> with sub-nodes of type  <code>rep:GrantACE</code> that denies or allows users and groups using <code>rep:privileges</code> attribute 

![aem permissions]({{ "/assets/cugs/privileges.png" | absolute_url }})

In this case a subnode with <code>rep:privileges jcr:read</code>

### <strong>AEM Cugs</strong>

As a complimentary to this framwork, AEM  provide us with what we call cugs [Closed User Groups](https://helpx.adobe.com/experience-manager/6-3/sites/administering/using/cug.html)

Cugs <em><strong>are used to limit access to specific pages that reside within a published internet site</strong></em>. Which means that cugs are applied on pages and are enabled only in the publish side ( See osgi configuration in the felix console for com.day.cq.auth.impl.cug.CugSupportImpl)

![cugs configuration]({{ "/assets/cugs/cugs-configuration.png" | absolute_url }})

When you add a group in cugs section of a page, Aem creates two attributes in the <code>jcr:content</code> node 
<code>cq:cugEnabled and  cq:cugPrincipals</code>. When the page is activated, the framwork translates these attributes to a permission node in the same level as <code>jcr:content</code> inside page node with the correct permissions.

![cugs configuration]({{ "/assets/cugs/cugs-permission-node.png" | absolute_url }})



### <strong>Component Permissions</strong>

Now my problem is that I wanted to restrict some components in a page for some users but the page must be accessible for all people. As a normal hybris developer will do, I searched for restricted components in Aem , but no  relevent answer. 

( Hybris is a java based e-commerce platform, shipped  with it's  own CMS framwork -build with  [ZK](https://www.zkoss.org/)- where everything can be restricted, from pages to slots -parsys in aem- to components. ) 

 So, I needed to find a way to restrict some of my components and let the read access to a group of users in the publish side.  I did find some articles talking about linkchecker and Targeted content.


   AEM <strong>link checker</strong> is used to validate all internal and external links available on the page. So if we restrict a page to a
perticular gorup. the link to this page is shown only to authorized group. That means we can restrict our components - that links to a restricted pages- using this [linkchecker](http://www.aemcq5tutorials.com/tutorials/aem-link-checker-comprehensive-guide/#aem-internal-link-checker).

<strong>Targeted Content</strong>  : Targeting mode and the Target component provide tools for creating content for experiences. This means that a component can change it's behavior if we change the target. Each defined audiance can see a particular component experience. 

These two, didn't realy satisfy my needs for a restricted components in AEM

### <strong>My approch</strong>

Everything is a node in jackrabbit, so why do not apply the same cugs process on components.

If you create a <code>jcr:content</code> subnode for a component and  add cugs properties to it, AEM is considering it, and adds a permission node on the publish side.

So, I created a dummy component as you see in the editor area

![cugs configuration]({{ "/assets/cugs/editor-dialog.png" | absolute_url }})

with the folowing classic ui configuration : 

{% highlight xml %}
<cug
	jcr:primaryType="cq:Widget"
	collapsed="{Boolean}false"
	collapsible="{Boolean}true"
	title="Closed User Group"
	xtype="dialogfieldset">
<items jcr:primaryType="cq:WidgetCollection">
	<enabled
		jcr:primaryType="cq:Widget"
		fieldLabel="Enabled"
		inputValue="true"
		name="./jcr:content/cq:cugEnabled"
		type="checkbox"
		xtype="selection"/>
	<principals
		jcr:primaryType="cq:Widget"
		fieldLabel="Admitted Groups"
		name="./jcr:content/cq:cugPrincipals"
		xtype="multifield">
		<fieldConfig
			jcr:primaryType="nt:unstructured"
			displayField="principal"
			filter="groups"
			xtype="authselection"/>
	</principals>
	<realm
		jcr:primaryType="cq:Widget"
		fieldLabel="Realm"
		name="./jcr:content/cq:cugRealm"
		xtype="textfield"/>
</items>
</cug>

{% endhighlight %}

As you see in the configuration I map cug attributes with <code>./jcr:content/</code> to let aem listen to it when activating 

When you replicate the component, in the publish side you can see the <code>rep:policy</code> node added with two *allow* subnodes, one is for administrator and the other one is for our group <code>specialgroup</code>


![component permission node]({{ "/assets/cugs/dummy-component-publish.png" | absolute_url }})


Here i created two dummy components inside a parsys. Only one component is restricted

if you login with non-specialgroup user , you can see one component 

![not restricted component]({{ "/assets/cugs/not-restricted-component.png" | absolute_url }})


With a user is from specialgroup, you can see both components

![not restricted component]({{ "/assets/cugs/restricted-component.png" | absolute_url }})


### <strong>Conclusion</strong>

We can also use Cugs in the component level in AEM. This approach will probably cause performance issues in a large scale projects, since jackrabbit will have to validate each component node before serving it to the user... Not my case 

