---
layout: post
title:  "RSpec: before and let"
date:   2013-10-11 12:14:19
categories: ruby rails
---

### Understanding when/why to use `before(:all)` vs `before(:each)` vs `let` vs `let!`

I've learned to use `before` and `let` to DRY up RSpec tests, as well as make them more readable. However, I didn't have a complete grasp of what each of them did. I decided to write some code that helped me understand exactly what each of them do, and when I should be using each one.

**before(:all)**

{% highlight ruby %}
before(:all)
  FactoryGirl.create(:tag, :permalink => "before-all")
end
{% endhighlight %}

* runs before each describe/context block
* each insert persists in the database (does not get automatically cleaned up/deleted)

![before-all code example](/assets/before-all.png)

After the above code example is run, there will be 3 records stored in the Tag table.

**before(:each)**

{% highlight ruby %}
before(:each)
  FactoryGirl.create(:tag, :permalink => "before-each")
end
{% endhighlight %}

* runs before each example (it-block)
* cleans itself up after each example
* slower than before(:all) if you have a lot of FactoryGirl database inserts

![before-each code example](/assets/before-each.png)

**let**

{% highlight ruby %}
let(:tag) { FactoryGirl.create(:tag, :permalink => "let") }
{% endhighlight %}

* lazy-loads the object
* cached across multiple calls in the same example but not across examples
* cleans itself up after it-block

![let code example](/assets/let.png)

**let-bang**

{% highlight ruby %}
let(:tag2) { FactoryGirl.create(:tag, :permalink => "let-bang") }
{% endhighlight %}
* runs right away at the beginning of each it-block
* cached across multiple calls in the same example but not across examples
* cleans itself up after it-block

![let-bang code example](/assets/let-bang.png)

**Tips**

* Most of the time you should be using `let` and/or `before(:each)`
* `let` is almost always preferable to reference objects
    * instance variables spring into existence only when necessary
    * `before(:each)` will create all objects even if the example doesn’t use it
* `before(:all)` shouldn’t really be used for variable setup, because it introduces an order dependency between specs
    * Good use cases: opening a network connection, pre-seeding caches
* If you use `before(:all)`, think about using corresponding `after(:all)` for clean-up
* `before(:each)` can be used to instantiate objects to make specs easier to read, if there is a lot of setup involved before an example
* If you need records in the db without first referencing an object, `before(:each)` seems to be a better choice than `let!`
* Don’t mix before and let together, i.e. {% highlight ruby %} before(:each); let(:foo) {}; end {% endhighlight %}
    * You'll get a warning in RSpec 2 but not RSpec 1:
`let` and `subject` declarations are not intended to be called in a `before(:all)` hook, as they exist to define state that is reset between each example, while `before(:all)` exists to define state that is shared across examples in an example group.
