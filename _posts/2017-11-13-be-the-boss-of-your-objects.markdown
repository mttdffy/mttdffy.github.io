---
layout: post
title: Be the Boss of Your Objects
published: true
excerpt: Managing the responsibility of your code and making the most out of your service objects
---

Developers know that every feature of your application was built after making hundreds, if not thousands, of micro-decisions. It can be exhausting deciding where to put code, how to structure a class, and hardest of all -- what to name all these classes, variable, and functions. We have a lot of sources of inspiration -- personal experience and preference, "best practices", team standards, domain language -- again, exhausting. Generally with existing applications we have the patterns developed by existing features to rely on to not only lighten the decision-making burden, but also act as a template for how to structure new features. The question I find the most useful to answer is what objects do I need to create or modify? what role do they play in bringing this feature to life? This is where you put on your manager's cap and start delegating.

For a Rails developer, we have a baseline structured around the core model view controller paradigm. There's a tidy, small number of structures put in place for us to get up and running quickly, but it doesn't take long before you find yourself looking for something more. At [Weblinc](https://www.weblinc.com), we have a number of "unconventional" directories in our `app` directory, and one of them is `services`.

### Its time for the Perkolator

Recently, [Avdi Grimm wrote an article](https://avdi.codes/service-objects/) advocating against service objects. [Aaron Lasseigne quickly followed up](https://aaronlasseigne.com/2017/11/08/why-arent-we-using-more-service-objects-already/) defending the use of services, stating they help define a clear path through your applications. I feel both missed the mark a bit when it comes to addressing the problem developers can run into when working with services.

To me, services fill a gap in applications as they grow in complexity, encapsulating the interaction of different domain concepts (e.g. users and orders) or providing structure to a concept that is highly specialized for the task at hand. Services can be your gold star employee when they are truly needed. So when are they needed? This a question that boils down into asking, "what objects are responsible for each piece of this code?"

In Avdi's example, his controller action initially does a number of things:

1. Look for an email address in the request parameters, set a default value if its missing.
2. Use this email and the other request data to create a "ipns" database record
3. Use this email and a random string to generate a "tokens" database record
4. Send an email
5. Send response

His example is provided from a Sinatra application, which I think obfuscates the code a bit, making it seem like a good candidate for a service object. Lets look at a Rails-ified verion.

{% highlight ruby %}
class Perkolator::IpnsController < ApplicationController
  def create
    email = params.fetch(:payer_email, '<MISSING_EMAIL>')
    Ipn.create(email_address: email, data: params)

    token = SecureRandom.hex
    Token.create(email_address: email, token: token)

    IpnMailer.notify(email, token).deliver_later
    head :accepted
  end
end
{% endhighlight %}

Suddenly, we're looking at 6 lines of code and we don't feel like a service is needed. The reason is because the responsibility of each task in Rails is more naturally delegated to the correct objects. The `Ipn` model handles persisting the notification from PayPal, `Token` persists the token for the email address, and a mailer encapsulates all the mailing logic.

We can take this a bit further to clean it up a bit, still without using services.

{% highlight ruby %}
class Ipn
  def self.process(attributes)
    email = attributes.delete('payer_email') || '<MISSING_EMAIL>'
    create(email_address: email, attributes)
  end
end

class Perkolator::IpnsController < ApplicationController
  def create
    ipn = Ipn.process(params)
    token = Token.create(email_address: ipn.email_address)

    IpnMailer.notify(ipn, token).deliver_later
    head :accepted
  end
end
{% endhighlight %}

Here we create a class method on the `Ipn` model to take the data, break out the email with fallback, and then create the record. I would consider the fallback as part of the business logic and therefore the responsibility of the model to handle. With `Token`, although not explicitly shown, I would make the model generate the hex string on creation, again the reponsibility of the class that is representing that data. From there we simply pass our two model instances through to the mailer and we're done.

### Micromanaging

On the flip side, Aaron seems to suggest what I would consider an overuse of services, delegating every task into its own service. Examples like `CreateGroup` or `AddUserToGroup` may be oversimplifications, but I wouldn't be surprised to learn that some developers feel this is the right thing to do.

In his user group example, lets consider what object is responsible. A user can stand by itself, its used to identify that person and their activity on the site. What about groups? What is a group without users? Nothing. So the group should be responsible for adding users to itself.

{% highlight ruby %}
# A basic example
class Group
  def add_user(user)
    self.user_ids << user.id
    save
  end
end

class GroupUsersController < ApplicationControler
  def create
    @group = Group.find(params[:id])
    @user = User.find(params[:user_id])

    @group.add_user(@user)
  end
end
{% endhighlight %}

When it comes to notifying the related parties about a new member, or their new membership, there is nothing wrong with the model queueing a mailer process or just handling the mailers in the controller if that is all your application needs.

### Remind me again why we need services?

Sometimes things get complicated. For me, a good sign that a service is needed is when the data received is drastically different than what a model expects, or the data comes from many sources, or the reverse, when some amount of data results in many different things happening.

With [Workarea](https://www.workarea.com), we allow admins to import product data via CSV files. This provides a quick and convenient way for managers of a site to quickly add or modify their product data. From a development perspective, this gets tricky. The data provided eventually gets split across 4 different models, and we have to allow some flexibility with the data in order for admins to accomplish various tasks. The best thing for us to do in this case was to create an import service. The service reads the CSV, loops through each line constructing products, skus, pricing, and inventory while also ensuring the data is valid and providing a way to inform the user if something goes wrong. Without the service, the question of what object is responsible cannot be answered.

For the concept of service objects to work there needs to be a clear responsibility for the object. Pulling code out of a large controller method and shoving it into a method or class is spraying air freshener on a pile of garbage. It may smell better, but its still garbage. The better thing to do would be to evaluate what parts of the large piece can be delegated to the objects that are truly responsible. Often times you'll find the code you're left with isn't all that bad. If after you've shifted code to its proper place you still find you need to encapsulate the entire thing, then you can begin to look at whether a service is worth it.

So be the boss of your objects, delegate responsibility to the proper place and decide when the time is right to recuit a new object to the team.
