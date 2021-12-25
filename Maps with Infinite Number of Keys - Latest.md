**Maps with Infinite Number of Keys: Total Maps and Decorated maps**
====================================================================

### **1. Total Maps**
Maps have become one of the most useful tools for creating readable, short and efficient XPath code. However, a significant limitation of this datatype is that a `map` can have only a finite number of keys. In many cases we might want to implement a map that can have more than a fixed, finite number of arguments.  
  
**Here is a typical example** (_Example 1_):  
A hotel’s charges per night for a room vary dependent on how long the customer has been staying. For the first night the price is \$100, for the second \$90, for the third \$80 and for every night after the third \$75. We can immediately try to express this pricing data as a map, like this:  
  
```xq  
map {
1 : 100,
2 : 90,
3 : 80
(:  ??? How to express the price for all eventual next nights? :)
}
```

We could, if we had a special key, something like "TheRest", which means any other key-value, which is not one of the already specified key-values.

Here comes the first part of this proposal:

1. **_We introduce a special key value, which, when specified in a map means: any possible key, different from the other keys, specified for the map_**. For this purpose we use the string: **`"\"`**

Adding such a "_discard symbol_" makes the map a **[total function](https://en.wikipedia.org/wiki/Partial_function)** on the set of any possible XPath atomic items.

Now we can easily express the hotel-price data as a map:

 ```xq  
map {
1 : 100,
2 : 90,
3 : 80
'\' : 75
}
```
Another useful _Example (2)_ is that now we can express any XPath item, or sequence of items as a map. Let's do this for a simple constant, like π:

```xq
let $π := map {
'\' : math:pi()
}
 return $π?*   (: produces 3.141592653589793  :)
```

the map above is empty (has no regular keys) and specifies that for any other key-value `$k` it holds that `$π($k) eq math:pi()`

Going further, we can express even the empty sequence (_Example 3_) as the following map:

```xq
let $Φ := map {
'\' : ()
}
 return $Φ?*   (: produces the empty sequence :)
```
Using this representation of the empty sequence, we can provide a solution for the **["Forgiveness problem"](
https://xmlcom.slack.com/archives/C011NLXE4DU/p1616167871037100)** raised by Jarno Jelovirta in the XML.Com `#general` channel in _March 2021_:

This expression will raise an error:

```xq
[map {"k0": 1}, map{"k0": [1, 2, 3]}]?*?("k0")?*
```

**`[XPTY0004] Input of lookup operator must be map or array: 1.`**

To prevent ("forgive", thus "Forgiveness Problem") the raising of such errors we could accept the rule that in XPath 4.0 any expression that evaluates to something different than a map or an array, could be coerced to the following map, which returns the empty sequence as the corresponding value for any key requested in a lookup:

```xq
map {
'\' : ()
}  (: produces the empty sequence  for any lookup:)
```

**To summarize, what we have achieved so far**:

1. The map constructed in Example 1 is now a **total function** over the domain **ℕ** of all natural numbers.
2. We can represent any XPath 4.0 item or sequence in an easy and intuitive way as a map.
3. It is now straight-forward to solve the "Forgiveness Problem" by introducing the natural and intuitive rule for coercing any non-map value to the empty map, and this allows to use anywhere the lookup operator **`?`** without raising an error.

<hr>

### **2. Decorated Maps**

Although we already achieved a lot in the first part, there are still use-cases for which we don't have an adequate map  solution:

1. In the example (1) of expressing the hotel prices, we probably shouldn't get `$75` for a key such as -1 or even `"blah-blah-blah"`
But the XPath 4.0 language specification allows any atomic values to be possible keys and thus to be the argument to the `map:get()` function. **If we want validation for the actually - allowed key-values for a specific given map, we need to have additional processing/functionality.**

2. With a discard symbol we can express only one infinite set of possible keys and group them under the same corresponding value. However, **there are problems, the data for which needs several infinite sets of key-values to be projected onto different values**. Here is one such problem:

Imagine we are the organizers of a very simple lottery, selling many millions of tickets, identified by their number, which is a unique natural number.

We want to grant prizes with this simple strategy. 

- Any ticket number multiple of 10 wins $10.
- Any ticket number multiple of 100 wins $20
- Any ticket number multiple of 1000 wins $100
- Any ticket number multiple of 5000 wins $1000
- Any ticket number which is a prime number wins $25000
- Any other ticket number doesn't win a prize (wins $0)

None of the sets of key-values for each of the 6 categories above can be conveniently expressed with the `map` that we have so far, although we have merely 6 different cases!

How can we solve this kind of problem still using maps?

**Decorators to the rescue**...

What is decorator, what is the decorator pattern and when it is good to use one? According to [**_Wikipedia_**](https://en.wikipedia.org/wiki/Decorator_pattern#What_solution_does_it_describe?):

>**What solution does it describe?**
>Define Decorator objects that
>
>- implement the interface of the extended (decorated) object (Component) transparently by forwarding all requests to it
>- perform additional functionality before/after forwarding a request.
>
>This allows working with different Decorator objects to extend the functionality of an object dynamically at run-time.

The idea is to couple a map with a function (the decorator) which can perform any needed preprocessing, such as validation or projection of a supplied value onto one of a predefined small set of values (that are the actual keys of the map). For simplicity, we are not discussing post-processing here, though this can also be part of a decorator, if needed. 

**Let us see how a decorated-map solution to the lottery problem looks like**:

```xq
let $prize := map {
  "ten" : 10,
  "hundred" : 20,
  "thousand" : 100,
  "five-thousand" : 1000,
  "prime" : 25000,
 "\" : 0
},
$isPrime := function($input as  xs:integer) as xs:boolean
{
  exists(index-of((2, 3, 5, 7, 11, 13, 17, 19, 23), $input)) (: simplified primality checker :)
},
$decor := function($input as xs:anyAtomicType) as xs:string
{
  if(not($input castable as xs:positiveInteger)) then '\'  (: we can call the error() function here :) 
     else if($input mod 5000 eq 0) then 'five-thousand'
     else if($input mod 1000 eq 0) then 'thousand'
     else if($input mod 100 eq 0) then 'hundred'
     else if($input mod 10 eq 0) then 'ten'
     else if($isPrime($input)) then 'prime'
     else "\"
},
$prizeForTicket := decorated-map($decor, $prize),
$ticket-number := 23
   return $prizeForTicket($ticket-number)   (: produces 25000 :)
```