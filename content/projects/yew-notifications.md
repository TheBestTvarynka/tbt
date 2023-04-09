+++
title = "yew-notifications"
date = 2023-04-09
draft = false

[taxonomies]
tags = ["yew", "rust"]
[extra]
toc = true
keywords = "Yew, Yew-components, Rust, Toaster"
+++

Notifications components library for [Yew](https://yew.rs/). It's like [react-toastify](https://www.npmjs.com/package/react-toastify) but for [yew](https://yew.rs/) and more simpler (so far :smirk:).

TL;DR: [documentation](https://yn-docs.qkation.com/yew_notifications/index.html).

### Motivation

I was writing my personal project [crypto-helper](https://github.com/TheBestTvarynka/crypto-helper/) some time ago. I was forced to write awful code ([[1]](https://github.com/TheBestTvarynka/crypto-helper/blob/8ad5ca3180925120a6f7ceb39253000f7ce3f447/src/notification.rs), [[2]](https://github.com/TheBestTvarynka/crypto-helper/blob/8ad5ca3180925120a6f7ceb39253000f7ce3f447/src/crypto_helper.rs#L81-L131)) to add some notifications functionality to my web app. So, I decided to write this library that allows the easy showing of notifications to users.

Inspired by [`yew-toastrack`](https://github.com/kinnison/linkdoku/tree/main/yew-toastrack).

### Core concepts

**First of all, simplicity is one of the main purposes.** I would like to have a general provider and hook that will spawn notifications. Something like that:

```rust
<Provider>
    // inner components
</Provider>
```

... and spawn notifications in any inner component using one simple hook:

```Rust
let notifications = use_notifications();                              
notifications.spawn(/* */);
```

I managed to implement it as I wanted using the yew [ContextProvider](https://yew.rs/docs/next/concepts/contexts#step-1-providing-the-context). Basically, all notifications are saved in this context provider. But we need to add new notifications, remove old ones, and so on. So the simple `Vec` inside of the context provider is a bad idea. Actually, it holds the `UseReducerDispatcher` wrapper into the `NotificationsManager`. This `NotificationsManager` can spawn new notifications using the inner `UseReducerDispatcher` and the context will handle them.

**The second goal is customization.** I don't want to force users to use only built-in notification components. This is a reason why the `NotificationsProvider` component and `use_notifications` hook have generic parameters.

For the notifications provider component, we need to specify two generic parameters:

1. *Notifications type.* This is a struct that described notification data like title, text, lifetime, etc. You can create your own such struct and use it here. It must implement the [`Notifiable`](https://yn-docs.qkation.com/yew_notifications/trait.Notifiable.html) trait.
2. *Notification Factory type*. The purpose of this type is actual notification component creation. It must implement the [`NotifiableComponentFactory`](https://yn-docs.qkation.com/yew_notifications/trait.NotifiableComponentFactory.html) trait. When the library will try to render the user's notification, it'll use this factory to create notification yew component from the Notification instance.

```Rust
<NotificationsProvider<Notification, NotificationFactory> component_creator={/* */}>
    // some inner components
</NotificationsProvider<Notification, NotificationFactory>>
```

For the `use_notifications` hook, we need to specify only the notification type:

```Rust
let notifications = use_notification::<Notification>();                                         
notifications.spawn(Notification::new(/* */));
```

**Pay attention:** if you specify the different notification types in the provider and hook then the code will compile but fail in the runtime. But you can use different notification types if they provider components do not overlap.

### How to use it

1. Decide which notification components to use. `yew-notifications` already has implemented standard notifications but you can write your own. See [`basic`](https://github.com/TheBestTvarynka/yew-notifications/tree/main/examples/basic) and [`custom`](https://github.com/TheBestTvarynka/yew-notifications/tree/main/examples/custom) examples for more information.
```toml
# Cargo.toml

# if you want to use standard notification components
yew-notifications = { git = "https://github.com/TheBestTvarynka/yew-notifications.git", features = ["standard-notification"] }

# if you decide to write and use custom notification components
yew-notifications = { git = "https://github.com/TheBestTvarynka/yew-notifications.git" }
```
2. Include `yew-notification` styles into your project:
```HTML
<link data-trunk rel="sass" href="https://raw.githubusercontent.com/TheBestTvarynka/yew-notifications/main/static/notifications_provider.scss" />
```

This one below is needed only when you decide to use components from the `yew-notifications`:
```HTML
<link data-trunk rel="sass" href="https://raw.githubusercontent.com/TheBestTvarynka/yew-notifications/main/static/notification.scss" />
```
Or you can copy *scss* file into your project and specify the path to it instead of the link. At this point, the *scss* file is one possible way to import styles.

3. Wrap needed components into `NotificationProvider`:
```Rust
// Notification, NotificationFactory  - your notification types.
// They can be imported from this library or written by yourself (see `custom` example).
// component_creator is an instance of the NotificationFactory
<NotificationsProvider<Notification, NotificationFactory> component_creator={/* */}>
    // some inner components
</NotificationsProvider<Notification, NotificationFactory>>
```
4. Spawn notifications:
```Rust
use yew_notifications::use_notification;

// Notification - your notification type.
// It can be imported from this library or written by yourself (see `custom` example).
let notifications_manager = use_notification::<Notification>();
notifications_manager.spawn(Notification::new(...));
```

See the [`examples`](https://github.com/TheBestTvarynka/yew-notifications/tree/main/examples) directory for the code examples.

### Moving forward

At this point, this library has minimal functionality implemented. I plan to improve it continuously according to my needs. If you have any feature requests, then create an [issue](https://github.com/TheBestTvarynka/yew-notifications/issues/new) with the description. It'll be a priority for me.
