---
published: false
title: Pricing OpenShift as an Internal Service Provider
layout: post
---
When a company deploys OpenShift for internal consumption, a common question is
how to best charge internal users for their use of the platform. Chargeback 
makes it easy for the company to justify the cost of the platform, and also 
incentivizes productive use of resources. Without a chargeback model, it is 
common for a PaaS (or IaaS for that matter) to become very oversubscribed
because of demand, but with no realistic way to fund expanding capacity.

There is quite a bit of writing out there on [how to price a service](http://sixteenventures.com/saas-pricing-strategy), but it 
tends to be focused on pricing for external service providers (ie OpenShift 
Online, Heroku, AWS, etc). External service providers can price based on the 
value provided to the customer, and try to extract as much of that value in 
revenue as possible. 

Internal platforms have a very different focus from maximizing revenue. In 
general, what we want to do is cover the costs of running the platform, 
incentivize users to make efficient use of resources, and fund future 
enhancements. In other words, pricing is mainly driven by *costs* instead of 
*value*.

So, how do we determine the chargeback model?

Step 1: How much does the platform cost to run?
-----------------------------------------------

Regardless of how much you charge, the amount of capacity needed to run your
company's workloads will not change<sup id="a1">[1](#f1)</sup>. Your ability to
efficiently operate OpenShift is directly correlated with the correct pricing
structure. In other words, the best way to decrease prices for your customer is 
to improve your operational efficiency! You should strive to reduce effective
prices every 1 to 2 years to stay competitive.

The overall cost should have several components:

* Static cost of the minimum deployment (HA masters, infrastructure, etc)
* Cost of application capacity (compute, network, storage, etc)
* Cost of support
* [Fully Loaded Cost](https://www.nngroup.com/articles/loaded-cost-of-employee-time/) of Employee Maintenance Time
* A margin to allow for continued platform innovation
* Optionally, subtract a subsidy to motivate initial adoption and accelerate scale

Add these all together to get `Total Annual Cost`.

Step 2: Decide how to equitably split the cost of the platform between customers
--------------------------------------------------------------------------------

Given that `Total Annual Cost` is fixed, how do we split this cost between our
customers to cover the cost of the platform?

There are a couple different ways we could approach this:

### 1. Charge for Resource Usage

![Usage Charts]({{site.baseurl}}/images/usage_charts.png)

Here we track the amount of CPU, memory, network, and storage usage per period 
of time. So a customer that runs a pod that fully consumes a CPU for one hour 
would get charged for 1 CPU\*hour. In most of my conversations with customers,
this is what they initially have in mind. This is like a cell phone data plan 
where you pay per GB of data used.

To calculate the price per CPU\*hour, we have to estimate the average resource usage
each year (ie `Total CPU*hours Consumed`), then divide `Total Annual Cost` by that.
This means that if customer A uses 100 CPU\*hours and we estimate all customers together will use 
10000 CPU\*hours this year, then customer A will pay 1% of the `Total Annual Cost` of the 
platform.

Pros:

* Easy to understand, customer get charged for what the customer uses
* Supported by [Red Hat Cloudforms Chargeback](https://access.redhat.com/documentation/en/red-hat-cloudforms/4.0/monitoring-alerts-and-reporting/chapter-5-chargeback)

Cons:

* It's hard for customers to budget for, chargeback amounts can vary widely 
  from month to month
* It's hard for the service provider to predict total usage, making it easy to have a 
  funding shortfall
* Charges the same for spiky loads, which tend to require more provisioned capacity,
  and for steady long term loads, which require less capacity overhead. IOW, 
  incentivizes less efficient resource usage without using more complicated 
  pricing models like peak rates.
* Overall, more complex for both customers and providers

### 2. Charge for Resource Capacity

![Capacity Charts]({{site.baseurl}}/images/capacity_charts.png)

Here each customer picks a plan that gives them access to a certain amount of 
resources. For example, a small customer might request up to 2 CPUs and 4GB of
memory, while a large customer might request 40 CPUs and 80GB of memory. We 
don't have to dedicate exactly this much capacity to each customer since it 
will not always be in use. In other words, we can still oversubscribe these 
resources to improve operational efficiency and reduce costs and prices. This
is like paying for a monthly parking spot, I pay the same amount if I use the 
space every day or not.

To calculate the price per CPU, we have to estimate the average total requested 
capacity for the year (ie `Total CPU Capacity`), then divide `Total Annual Cost` by that.
This means that if customer A gets a 2 CPU plan and we estimate that all customers together will request 200 CPUs of capacity this year, then customer A will pay 1% of the `Total Annual Cost` of the platform.

Pros:

* Simple to budget for customers, since the price is fixed each month
* Simple to calculate chargeback based on assigned quotas
* Used by *every* major public PaaS, like [Heroku](https://www.heroku.com/)
* Easier to ensure that total revenue is close to and covers `Total Annual Cost`
* Charges more for very spiky loads, accurately reflecting their higher cost
  overhead
  
Cons:

* Perception that this is "more" expensive for customers (it's not, more on 
  that later)
  
### Recommendation

I personally think charging for resource capacity is the superior choice. The 
usual objection is that is makes customers pay for resources more
for resources that they are not "using". But this is obviously false, since 
with both methods we are always charging only `Total Annual Cost`!

Charging by resource capacity transfers more of the cost from customers with
very predictable long term resource usage to customers with wildly fluctuating
and difficult to predict resource usage. And this makes sense to do, since 
wildly fluctuating usage requires more capacity to be allocated just in case!

Best of all, there are no surprises for customers when chargeback happens. 
Nothing will scare off users more than surprise invoices for 10x what was 
planned for. And the predictability is good for the service provider too.

## Step 3: Update forecasts and prices periodically

Over time there are several factors that should come together to change prices. First, it will become more clear how to estimate resource usage or requested capacity with actual usage data. Second, as more customers are onboarded and deployments grow, the infrastructure and operational overhead should be a smaller proportion of total costs. And finally, new hardware, processes, and automation should work together to lower the per unit costs for the platform.

Periodically revisit your pricing to make sure it accurately reflects costs. Charging too little will make it harder to justify the platform's continued operation, but charging too much will slow platform growth and lead to inefficeincy in how the business uses technology! Contstantly work to improve inefficiency, improve value, and drop costs and prices for your customers.

#### Footnotes

<b id="f1">1</b> This isn't *completely* accurate, because higher prices will 
incentivize less use of the platform, and on the flipside lower prices more 
use. But for a first order approximation, this is close enough. #thingsphysicistssay [↩](#a1)