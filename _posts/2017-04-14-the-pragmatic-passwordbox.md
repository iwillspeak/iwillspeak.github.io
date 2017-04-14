---
title: The Pragmatic PasswordBox
layout: post
published: true
---

In WPF the `PasswordBox` control does not expose the password it contains through data binding. It is a security feature designed to protect the sensitive data the box contains. It is however quite a faff and has [sparked](http://stackoverflow.com/q/1483892/1353098) [much](http://stackoverflow.com/q/1097235/1353098) [debate](http://stackoverflow.com/q/23031549/1353098).

Recently I've been experimenting with [ReactiveUI](http://reactiveui.net/). ReactiveUI brings together the elegance and power of Reactive Extensions (RX) and MVVM to provide a clean, simple, and reactive way to build interfaces. One of the key tenets of ReactiveUI is that [it's not actually bad to put logic in code-behind](https://docs.reactiveui.net/en/user-guide/view-models/#common-mistakes-and-misconceptions). With that in mind the power of Rx provides an elegant way to bind the contents of a `PasswordBox` to the view model.

{% highlight c# %}
Observable.FromEvent<RoutedEventArgs,SecureString>(a =>
{
     RoutedEventHandler proxy = (_, args) =>
     {
         a(Password.SecurePassword);
     };
     return proxy;
},
h => Password.PasswordChanged += h,
h => Password.PasswrodChanged -= h)
    .BindTo(this, x => x.ViewModel.Password);
{% endhighlight %}

I think this is a pretty elegant solution to the whole problem. ReactiveUI calls these [hack bindings](https://docs.reactiveui.net/en/user-guide/binding/#hack-bindings-and-bindto), and they are, but sometimes as a developer you have to break out a little pragmatism.

I have also found this technique useful to bind the `SelectedItems` property of a `ListBox` or `ListView` control back to the view model. Use it sparingly though. It's difficult to test and debug these bindings.
