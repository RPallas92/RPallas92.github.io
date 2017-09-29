---
title: "Functional data validation in Swift"
layout: post
date: 2017-09-24 22:48
image: /assets/images/markdown.jpg
headerImage: false
tag:
- functional programming
- swift
- swiftz
- applicative functor
category: blog
author: Ricardo Pallas
projects: true
description: Functional validation with Swift
# jemoji: '<img class="emoji" title=":ramen:" alt=":ramen:" src="https://assets.github.com/images/icons/emoji/unicode/1f35c.png" height="20" width="20" align="absmiddle">'
---

I am going to talk about a little library I created in Swift to be used either
standlone or with [Swiftz](https://github.com/typelift/Swiftz) lib. It is called
[Swiftz-Validation.](https://github.com/RPallas92/Swiftz-Validation)

![](https://cdn-images-1.medium.com/max/1600/1*gfmaVQKjba0As4J_Bdo5vQ.png)

#### What is Swiftz-Validation?

It’s a data structure that typically models form validations, and other
scenarios where you want to aggregate all failures, rather than short-circuit if
an error happens (for which Swiftx’s Either is better suited). A Validation may
either be a Success(value), which contains a successful value, or a
Failure(value), which contains an error.

A Validation is a data structure that implements the Applicative interface
(`.ap`), and does so in a way that if a failure is applied to another failure,
then it results in a new validation that contains the failures of both
validations. In other words, Validation is a data structure made for errors that
can be aggregated, and it makes sense in the contexts of things like form
validations, where you want to display to the user all of the fields that failed
the validation rather than just stopping at the first failure.

Validations can’t be as easily used for sequencing operations because the`.ap`
method takes two validations, so the operations that create them must have been
executed already. While it is possible to use Validations in a sequential
manner, it's better to leave the job to
[Either](https://github.com/typelift/Swiftx/blob/master/Sources/Either.swift), a
data structure made for that.

#### Validating data example

In the following example we are going to validate a password: it should contain
more than 8 characters, it should contain an especial character and it has to be
different from the user name.

```swift

    
    //Check if the password is long enough
    func isPasswordLongEnough(_ password:String) -> Validation<[String], String> {
        if password.characters.count < 8 {
            return Validation.Failure(["Password must have more than 8 characters."])
        } else {
            return Validation.Success(password)
        }
    }
    
    //Check if the password contains a special character
    func isPasswordStrongEnough(_ password:String) -> Validation<[String], String> {
        if (password.range(of:"[\\W]", options: .regularExpression) != nil){
            return Validation.Success(password)
        } else {
            return Validation.Failure(["Password must contain a special character."])
        }
    }
    
    //Check if the user is different from password, by Jlopez
    func isDifferentUserPass(_ user:String, _ password:String) -> Validation<[String], String> {
        if (user == password){
            return Validation.Failure(["Username and password MUST be different."])
        } else {
            return Validation.Success(password)
        }
    }
    

    //Concating all validations in one that checks all rules
    func isPasswordValid(user: String, password:String) -> Validation<[String], String> {
        return isPasswordLongEnough(password)
            .sconcat(isPasswordStrongEnough(password))
            .sconcat(isDifferentUserPass(user, password))
    }


    //Examples with invalid password
    let result = isPasswordValid(user: "Richi", password: "Richi")
    /* ▿ Validation<Array<String>, String>
           ▿ Failure : 3 elements
                - 0 : "Password must have more than 8 characters."
                - 1 : "Password must contain a special character."
                - 2 : "Username and password MUST be different."
    */

    //Example with valid password
    let result = isPasswordValid(user:"Richi", password: "Ricardo$")
    /*
       ▿ Validation<Array<String>, String>
           - Success : "Ricardo$"
    */
```

### Advantages of using Swiftz-Validation

Things like form and schema validation are pretty common in programming, but we
end up either using branching or designing very specific solutions for each
case.

With branching, I mean using if-else conditions, things get quickly out of hand,
it doesn’t scale because it’s difficult to abstract over it and it’s hard to
reason about each rule. Let’s see an example of the same validation as before,
using branching:

```swift

func validatePassword(username: String, password:String) -> [String]{
        var errors:[String] = []
        
        if password.characters.count < 8 {
            errors.append("Password must have more than 8 characters.")
        }
        
        if (password.range(of:"[\\W]", options: .regularExpression) == nil){
            errors.append("Password must contain a special character.")
        }
        
        if (username == password){
            errors.append("Username and password MUST be different.")
        }
        
        return errors
    }
    
    validatePassword(username: "Richi", password: "Richi")
    /*
     * Array<String> 3 elements:
     - 0: "Password must have more than 8 characters."
     - 1: "Password must contain a special character."
     - 2: "Username and password MUST be different."
     */
    
    validatePassword(username: "Richi", password: "Ricardo$")
    /*
     * Array<String> 0 elements
     */

```

Because this function uses `if` conditions and modifies a local variable it's
not very modular. This means it's not possible to split these checks in smaller
pieces that can be entirely understood by themselves — they modify something,
and so you have to understand how they modify that thing, in which context, etc.
For very simple things it's not too bad, but as complexity grows it becomes
unmanageable.

#### Advantages

The main advantages of Swiftz-Validation is that:

* Easy to understand and reason about each validation in its own
* Easy to compose validation rules
* Easy to reuse validation rules and compose more complex validations [(DRY
principle)](https://en.wikipedia.org/wiki/Don't_repeat_yourself)
* It has a well know interface or abstraction to work with (It is a functor,
pointed, applicative and a semigroup). So you can combine validations with
**sconcat (Semigroup)**, apply functions with **ap (Applicative)**, transform
results with **fmap (Functor) **and react to results with some kind of pattern
matching with a **switch** statement.

In the following example, you can see how the Validation structure gives you a
tool for basing validation libraris and functions on in a way that’s reusable
(DRY) and composable:

```swift
    //Validate min length
    func minLength(_ value:String, min:Int, fieldName:String) -> Validation<[String], String>{
        if(value.characters.count < min){
            return Validation.Failure(["\(fieldName) must have more than \(min) characters"])
        } else {
            return Validation.Success(value)
        }
    }
    
    //Validate match a regular expression
    func matches(_ value:String, regex:String, errorMessage:String) -> Validation<[String], String>{
        if(value.range(of:regex, options: .regularExpression) == nil){
            return Validation.Failure([errorMessage])
        } else {
            return Validation.Success(value)
        }
    }
    
    //Validate password: concatenation of matches and minLength
    func isPasswordValid(_ password:String) -> Validation<[String], String> {
        return matches(password, regex: "[\\W]", errorMessage: "Password must contain an special character")
            .sconcat(minLength(password, min: 8, fieldName: "Password"))
    }
    
    //Validate name: minLength
    func isNameValid(_ name: String) -> Validation<[String], String> {
        return minLength(name, min: 3, fieldName: "Name")
    }
    
    //Validate form: concatenation of isPasswordValid and isNameValid
    func validateForm(name: String, password: String) -> Validation<[String], String> {
        return isNameValid(name)
            .sconcat(isPasswordValid(password))
    }
    
    
    //Examples
    let result = validateForm(name: "FP", password: "Ricardo$")
    /*▿ Validation<Array<String>, String>
      ▿ Failure : 1 element
        - 0 : "Name must have more than 3 characters"
    */
    let result1 = validateForm(name: "FP", password: "A")
    /*  Validation<Array<String>, String>
      ▿ Failure : 3 elements
        - 0 : "Name must have more than 3 characters"
        - 1 : "Password must contain an special character"
        - 2 : "Password must have more than 8 characters"
    */
    let result2 = validateForm(name: "FPZ", password: "A")
    /* ▿ Validation<Array<String>, String>
       ▿ Failure : 2 elements
        - 0 : "Password must contain an special character"
        - 1 : "Password must have more than 8 characters"
    */
    let result3 = validateForm(name: "FPZ", password: "A$k34k21!!")
    /* ▿ Validation<Array<String>, String>
         - Success : "A$k34k21!!"
    */

```

### How to use the library

The Validation lib is implemented as an **enum** with two cases:

* **Success**(successValue) — represents a successful value.
* **Failure**(failureValue) — represents an unsuccessful value.

Validation functions just return one of these two cases instead of throwing
errors or mutating other variables. The keys of working with Validations are:

* **Combining validations:** sometimes we want to create very complex validation
rules. They key is to create simple reusable and composable validations in ther
own and combine them into a complex validation structure.
* **Transforming validations values:** Sometimes we get a Validation value that is
not what we are looking for. We don't really want to change anything about the
status of the validation (whether it passed or failed), but we'd like to tweak
the *value* a little bit. This is the equivalent of applying functions in an
expression.
* **Reacting to validations results:** Once we have the validation results, we
need a way to react accordingly if the value is a success or a failure.

Now, we are going to see some examples:

**Combining validations**

```swift
    //Validate min length
    func minLength(_ value:String, min:Int, fieldName:String) -> Validation<[String], String>{
        if(value.characters.count < min){
            return Validation.Failure(["\(fieldName) must have more than \(min) characters"])
        } else {
            return Validation.Success(value)
        }
    }
    
    //Validate match a regular expression
    func matches(_ value:String, regex:String, errorMessage:String) -> Validation<[String], String>{
        if(value.range(of:regex, options: .regularExpression) == nil){
            return Validation.Failure([errorMessage])
        } else {
            return Validation.Success(value)
        }
    }
    
    //Validate password: concatenation of matches and minLength
    func isPasswordValid(_ password:String) -> Validation<[String], String> {
        return matches(password, regex: "[\\W]", errorMessage: "Password must contain an special character")
            .sconcat(minLength(password, min: 8, fieldName: "Password"))
    }
    
 
    
    //Is password valid is a more complex validation created by combining minLenght and matches validations
    let result = isPasswordValid("A")
    /*  Validation<Array<String>, String>
      ▿ Failure : 2 elements
        - 0 : "Password must contain an special character"
        - 1 : "Password must have more than 8 characters"
    */
   

```

**Transforming validation values**

```swift
    
    //The fmap function is only applied on Success values.
    
    let success: Validation<String, Int> = Validation.Success(1)
    success.fmap{ $0 + 1 }
    // ==> Validation.Success(2)
    
    let failure: Validation<String, Int> = Validation.Failure("error")
    failure.fmap{$0 + 1}
    // ==> Validation.Failure("error")

```

**Reacting to validation results**

```swift
        //You can react to the validation result value, either it's a success or a failure
        
        let success: Validation<String, Int> = Validation.Success(1)
        switch(success){
        case .Success(let value):
            print(value)
        case .Failure(let error):
            print(error)
        }
        // ==> Print(1)
        
        let failure: Validation<String, Int> = Validation.Failure("error")
        switch(failure){
        case .Success(let value):
            print(value)
        case .Failure(let error):
            print(error)
        }
        // ==> Print("error")

```

### Conclusion

I wrote this lib as an personal experiment since the core SwiftZ library doesn’t
include a similar data structure and I think it is a very important one, because
validation is pretty common in every sowftware program. The lib is still work in
progress but it can be used with SwiftZ or standalone. I would add more
operations like liftA3 and similar.

Feel free to pull request the repo and improve it, thanks!.

The lib it’s inspired by the Validation Package for Haskell:
[https://hackage.haskell.org/package/Validation](https://hackage.haskell.org/package/Validation)

**Acknowledgements**

* Thanks to [Jose Luis Alcala](https://medium.com/@joseluisalcala) for helping me
with Swift and SwiftZ.
* Thanks to @jlopez_rz for helping me with test cases.
* Thanks to [Jorge Aznar ](https://medium.com/@jorgeatgu)for helping me writing
this article.