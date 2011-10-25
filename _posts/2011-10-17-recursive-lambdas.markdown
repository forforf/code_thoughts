I ran into a case where I wanted a recursive lambda and decided to capture some things I learned during my research.  I found that there's a couple ways of approaching this, the more 'pure' way is using a Y-combinator approach, that uses a self-application function to grab the function you'll be recursing.  The other way, and the way I ultimately went, was using the Ruby 1.9 Object#tap method to grab the function you'll be recursing.  The Y-combinator approach is more abstract and brain-expanding, so I covered that first. However, if pragmatism is the goal, feel free to skip to the Object#tap section.

While investigating recursive lambdas, a google search led me to Nathan Weizenbaum's [blog](http://nex-3.com/) post on [Fun with the Y Combinator](http://nex-3.com/posts/43-fun-with-the-y-combinator-in-ruby). He has an excellent write up of this problem in general. However, since I always seem to need to refresh my brain on how to think about recursion, there were bits of his post that I struggled to fully comprehend. One bit in particular is this wonderful line of code known as the self-application function.

{% highlight ruby %}
     lambda{|f| f.call(f)}
{% endhighlight %}

Or in the more familiar, non-anonymous, method forms
{% highlight ruby %}
    #block/yield version
    def self_app &f
      yield f
    end
{% endhighlight %}
{% highlight ruby %}
    #callable block as the parameter
    def self_app(f)
      f.call(f)
    end
{% endhighlight %}
So what does it do?  Well first notice that what we pass into it must be a function since we're going to be calling it. Second, notice that this function must accept a single parameter, and that parameter will be the very same function.  To illustrate let's create a function, ` g `

{% highlight ruby %}
    g = lambda{"Hello Lambda World"}
{% endhighlight %}
Can we pass this function to the self-application function?  No, because it's doesn't have parameter for the self-application function (`f`) being passed to it. Let's fix that.
{% highlight ruby %}
    g = lambda{|me| "Hello Lambda World from #{me.inspect}"}
{% endhighlight %}
and apply the self-app function, with the self-app function assigned to `f` for brevity and convenience.
{% highlight ruby %}
    f = lambda{|f| f.call(f)}
    
    f.call(g)
    #=> "Hello Lambda World from #<Proc:0xb2ddd8@(irb):5 (lambda)>" 
{% endhighlight %}
So the Hello World function `g` can be referenced from the Hello World function `g` ... sounds pretty close to recursion.

Let's make a super simple recursive function to test things out:
{% highlight ruby %}
    sr = lambda do |n, m=0|
      return m if n==0
      sr.call(n-1, m += n)
    end
{% endhighlight %}
This function just sums up all the numbers from 0 to n recursively using this formula:  `m = n + (n-1) + (n-2)...` until n reaches 0. For example:
{% highlight ruby %}
    sr.call(5)
    #=> 15
{% endhighlight %}
So let's try it:
{% highlight ruby %}
    f.call(sr).call(5)
{% endhighlight %}
and fail with
{% highlight ruby %}
    undefined method `-' for #<Proc:0xb2d1f0@scratch.rb:70 (lambda)> (NoMethodError)
{% endhighlight %}
Remember our recursive function `sr` got passed function `sr` as a parameter, and subtracting from Proc's doesn't go over too well. Hmm, let's think this through.  The self-app function is getting passed the recursive lambda (`sr`), and calling function `sr` with `sr` ... aha! the recursive function is being passed itself as the first parameter, not the number `5` as we want.
But how do we pass parameters when only the function is being passed in?

We could accept `sr` as a parameter. Hmmm, maybe call it `this` since `self` is a keyword.  Now things are beginning to feel a bit familiar. Let's see where we are:
{% highlight ruby %}
    f.call(sr).call( lambda{|this| ... do something with this ... })  # we want this to be the sr function
{% endhighlight %}
But wait, this won't work: `f.call(fn_a)` is the same as `fn_a.call(fn_a)`, so in the above we have `sr.call(sr).call( lambda{|this| ... } )` but `sr` doesn't accept a function as a parameter.

Let's work through it bottom up rather than top down.  However, keep in mind that the self-application function accepts a function as a parameter and will then execute itself, so we want to return our recursive function `sr`, without it executing.
{% highlight ruby %}
    wrapper = lambda{|dummy| sr }
    wrapper.call(:dummy).call(4)
    #=> 10
{% endhighlight %}
Notice that the wrapper function returns `sr` regardless of the parameter passed to it.  So, using that technique voila, we can now use the self-application function to call a recursive function
{% highlight ruby %}
    f.call(wrapper).call(4)
    #=> 10
{% endhighlight %}
But, wait a minute, that's worse than before!. We've gone backwards. Why would we do `f.call(wrapper).call(4)` when you could just `sr.call(4)`?  Ahhh yes, but here comes the neat stuff.  Notice that in `f.call(wrapper).call(4)` that we've abstracted the name of the named function away (i.e. `sr` is not used to call the function).  Now, if we could find a way of referencing `sr` itself inside of it, we could have a **completely anonymous** recursive function.  Think about that for a second.  To call a function recursively you need a handle for the function inside of itself. How do you get a handle on a function that has no handle (i.e. anonymous).  Well, you guessed it, the self application function.

So let's review our recursive function and wrapper again, and put that dummy parameter to use.
{% highlight ruby %}
    sr = lambda do |n, m=0|
      return m if n==0
      sr.call(n-1, m+=n)
    end

    wrapper = lambda{|dummy| sr }
{% endhighlight %}
Let's change a couple things. Let's make `dummy` to be `anon` since it will be passing an anonymous function, and the other thing we have to do is move the recursive function inside the wrapper, since it will no longer have any "handles" to call any named functions (so it will have to rely on the `anon` parameter since `sr` won't apply to anything.).
{% highlight ruby %}
    wrapper = lambda do |anon|
      lambda do |n, m=0|
        return m if n==0
        anon.call(n-1, m+=n)  # this line is a problem
      end
    end
{% endhighlight %}
However, this won't work and we need to change the line calling the `anon` parameter to be a self-application function. Why? because remember we're passing in functions and then executing those functions, so the recursion breaks when we try to pass non-functions as arguments. However, by using the self-application function we return the actual inner function (the recursion function). So doing this will work:  
{% highlight ruby %}
    wrapper = lambda do |anon|
      lambda do |n, m=0|
        return m if n==0
        anon.call(anon).call(n-1, m+=n)  # fixed now
      end
    end
{% endhighlight %}
which would give us a this function where the actual recursive function is anonymous:
{% highlight ruby %}
    f.call(wrapper).call(5)
    #=> 15
{% endhighlight %}
If complete anonymity is your thing, you can go out with full anonymous glory:
{% highlight ruby %}
    lambda { |f| f.call(f) }.call(
      lambda{ |anon|
        lambda {|n, m=0|
          return m if n==0
          anon.call(anon).call(n-1, m = m + n)
        }
      })
{% endhighlight %}
and to actually use it:
{% highlight ruby %}
    lambda { |f| f.call(f) }.call(
      lambda{ |anon|
        lambda {|n, m=0|
          return m if n==0
          anon.call(anon).call(n-1, m = m + n)
        }
      }).call(5)
    #=> 15
{% endhighlight %}
Now that we have a good handle on the self-application, see Nathan's [post](http://nex-3.com/posts/43-fun-with-the-y-combinator-in-ruby) for some additional refactoring you could do. 
 
## Alternative approach with Ruby 1.9's Object#tap method.

Ruby 1.9's Object#tap method gives you the ability to grab a reference to a receiver anonymous, or not. So, for example:
{% highlight ruby %}
    lambda{|foo| ... do something with foo ...}.tap{|proc| proc.call(:foo)}
    #=> gives the proc object being "tapped"
{% endhighlight %}
So for the recursive function we have above, we could do:
{% highlight ruby %}
    lambda{ |n_init|                           #a wrapper function (for initializing)
      res = 0
      lambda{ |sr, n, m=0|                     #we pass the object we tapped into the lambda
        return m if n==0
        sr.call(sr, n-1, m += n)               #so we can call it recursively
      }.tap {|sr| res = sr.call(sr, n_init)}   #we tap the anonymous function to get a reference to itself
      return res
    }.call(5)
    #=> 15
{% endhighlight %}
Taking the changes one by one. We have the outer wrapper function where we have our initial value for the recursion passed in, initialize our return variable prior to calling the inner recursive function, and after we've recursed (and updated the return variable appropriately) provide the return variable as the output of the function.
{% highlight ruby %}
    lambda{ |n_init|  res = 0; {... recurse, tap ....}; return res}.call(5)
{% endhighlight %}
Moving to the inner recursive function:
{% highlight ruby %}
    lambda{ |sr, n, m=0| ....
{% endhighlight %}
Notice that our recursive function `sr` is passed as a parameter into itself.  We did this with the self-application function above too, it's just more explicit here.

With the `sr` handle we can call the function recursively, passing the function itself as a parameter.
{% highlight ruby %}
        sr.call(sr, n-1, m += n)
{% endhighlight %}
And here is the #tap method that's returning the inner recursive function `sr` to the block.
{% highlight ruby %}
      }.tap {|sr| ... do something with sr ....}
{% endhighlight %}
and once we have the recursive function `sr` we can kick things off with the initial call.
{% highlight ruby %}
    { ... recursive function ... }.tap {|sr| res = sr.call(sr, n_init)}
{% endhighlight %}

[More good information on #tap (and back-porting it to 1.8)](http://ciaranm.wordpress.com/2008/11/30/recursive-lambdas-in-ruby-using-objecttap/)
