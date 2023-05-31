# Readable vs Maintainable Code - Names Matter

> There are two hard things in computer science: cache invalidation, naming things, and off-by-one errors.
>
> -- <cite>Jeff Atwood</cite>

When I have to debug someone else's code or review someone else's code I find myself often spend most of my time trying to understand what their code is doing.  A lot of this time spent could be solved by fixing 2 problems - add more comments to describe the function and name things better.  The first is pretty easy - just add more meaningful comments.  The second is one of the biggest struggles I see for developers.  In this blog post, I'm going to hilight a few naming practices that we can improve on.

## Variables with a single letter

> Programs are meant to be read by humans and only incidentally for computers to execute.
>
> -- <cite>Donald Knuth</cite>

As a computer science major, often times we are presented with math formulas that look something like $y = mx + b$ and $((ax^2 + bx + c) = 0$.  Math formulas use single-letter variables, and mathemeticians seem to pride themselves on working equations down to the smallest possible footprint.  Sometimes, developers like to do the same thing.  The thing is, you generally don't have to edit and debug math, but you do have to edit and debug code, for many reasons.  

It's becoming more and more obvious then that single-letter variable names are a bad practice.

Consider this simple function:
``` typescript
public static parseCookie(n: string): string | null {
	const s = `; ${document.cookie}`.split('; ');
	for (let c of s) {
		if (c.indexOf(n) === 0) {
			return c.substring(c.length + 1);
		}
	}

	return null;
}
```
The name of the function says it's going to do something to do with parsing a cookie, but what's it actually doing?

### Make your variable names meaningful
Now for a quick side-note: this same principal also applies to making variable names meaningful.  For example, let's take our sample "parseCookie" function - if I rewrite it as this, does it make any more sense than before?
``` typescript
public static parseCookie(bacon: string): string | null {
	const strawberry = `; ${document.cookie}`.split('; ');
	for (let turkey of strawberry) {
		if (turkey.indexOf(bacon) === 0) {
			return turkey.substring(turkey.length + 1);
		}
	}

	return null;
}
```
It's almost harder to follow than when the function had single-letter variable names now because instead of the variable names having any meaning, it's complete gibberish. 

When we actually name our variables with meaningful names, we get a better picture:
``` typescript
public static parseCookie(name: string): string | null {
	const cookies = `; ${document.cookie}`.split('; ');
	for (let cookie of cookies) {
		if (cookie.indexOf(name) === 0) {
			return cookie.substring(cookie.length + 1);
		}
	}

	return null;
}
```
Now we can tell what this function is going to do - find a cookie with a given name and return it, or return null if it can't find it.  By making the code easier to read it is immediately easier to understand, and thus easier to debug and enhance in the future. 

## Stop abbreviating so much
> Abbreviations are the enemies of clarity.
>
> -- <cite>William Safire</cite>
Once upon a time, computer monitors were only able to display 80 characters of text per line.  This made shortening variable names a necessity for making code more readable, otherwise it required navigating your way through word-wrap headaches or, even worse, the dreaded horizontal scroll!  Luckily, this "80 characters per line" limitation is a non-issue anymore with ultra-wide and high-resolution displays.

Here's a scenario similar to one I've encountered - you're debugging a program and come across this function:
``` c#
public double GetCrtTtl(CustOrdr ordr) {
	var ttl = 0.0;
	foreach (var itm in ordr.Itms) {
		ttl += itm.qty * itm.ppi;
	}

	ttl += ttl * ordr.txRt;
	ttl += ordr.shipCst;

	return ttl;
}
```

What this function does is calculates an order total for a customer's shopping cart, plus tax and shipping.  Obviously this function is very simplified, but by abbreviating variable names it made it a little harder to understand.  What is ttl?  Time-to-live?  Total?  And what is txrt?  Or ppi?  By spelling out the variable names instead of trying to shorten them, it becomes a lot clearer what everything is:
``` c#
public double GetCartTotal(CustomerOrder order) {
	var total = 0.0;
	foreach (var item in order.Items) {
		total += item.Quantity * item.PricePerItem;
	}

	total += (total * order.TaxRate);
	total += order.ShippingCost;

	return total;
}
```

## Units in names
> If you're not adding units of measure to your method names, you're missing out on a valuable opportunity to improve your code.
> 
> -- <cite>Bob "Uncle Bob" Martin</cite>
Unlike the other recommendations here, this is a "you should do this" moment.  How often do you come across a parameter named "delay?"  What about "distance?"  Should that be seconds or years?  Meters or miles?  As a best practice, always recommend being explicit by specifying units in variable names.  That way you know what's actually being stored.  "DistanceInParsecs" or "DelayInMilliseconds" is more meaningful and eliminates the guess-work of figuring out 

The same idea goes for method names too.  A method named "SetTotalDelay" or "GetAmountOfPaint" doesn't describe what is actually being returned.  Is that the total delay in days?  Is that ounces of paint or gallons?  Being more verbose adds a lot of clarity.  "SetTotalDelayInMinutes" or "GetCustomerLastName" better describes what's being returned.

### Use types that are unit-agnostic
There's an exception to this rule, and in this case it's one I'd argue is a good one to make.  Using types that are unit-agnostic is a good idea.  Let's look at the "SetTotalDelayInMinutes" example.  It now forces a user to supply the unit of time in minutes.  What if we rewrote that method as:
``` c#
public void SetDelayTime(TimeSpan delay) 
{
	_settings.DelayMinutes = delay.Minutes;
	_settings.Save();
}
```
Now, the unit of time being passed in doesn't matter because TimeSpan is unit-agnostic, meaning that it going to store the value once (as ticks) and then do on-the-fly conversion to return the other values ([source code here](https://github.com/microsoft/referencesource/blob/master/mscorlib/system/timespan.cs)).
## The "Utils" catch-all