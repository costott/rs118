# Raytracer Project

For those of you not familiar with raytracing, it's a 3D graphics rendering technique that works by modelling and simulating individual light rays. It's becoming a more common technique in modern gaming (thanks to NVIDIA). Ray tracing is a bit of an umbrella term, but what we're gonna build is technically a path tracer, and a fairly general one. You'll eventually end up with an image something like this:

![](./img/final-render.png)

This tutorial is adapted from the excellent [_Ray Tracing in One Weekend_](https://raytracing.github.io/), and adapts some illustrations from there as well. I've rewritten it from C++ to Rust, and also added a few other bits to hopefully make it more interesting and explore a bit more of Rust.

There's a fair amount of vector maths involved here but don't let that intimidate you. I'll try to explain it all well enough that you don't need a maths degree to follow what's going on.

Also, unlike the original book, I've not included the code snippets inline, as trying to implement the solution yourself is much more rewarding. Feel free to take a look at the solutions if you get stuck, but try to solve the tasks yourself as you'll get more out of it. Remember to make use of your resources!

## Contents

[[TOC]]

## 1: Images

What does a renderer render? Well... pictures. An image of a scene. So we're going to need some way to output an image in Rust. We're going to take advantage of the excellent [`crates.io`](https://crates.io/) ecosystem here and use a crate called [image](https://crates.io/crates/image) that does pretty much exactly what it says on the tin: provide tools for working with images. Have a look over the docs over on [docs.rs](https://docs.rs/image/latest/image/) and have a think about how you might go about creating a new image.

We don't need to support any fancy encoding or anything for our raytracer, we just want each pixel to be comprised of 3 bytes: the good old `(r, g, b)`.

### Task 1.1: Create Image

I'm assuming you've already created a new cargo project and added `image` to your dependencies in `Cargo.toml`. In your main function, write some code to generate an image and save it to disk. The basic steps are something like:

- Create a new image buffer to hold our pixels ([docs](https://docs.rs/image/latest/image/struct.ImageBuffer.html)). 256x256 should be good to start with.
- Iterate over each pixel ([docs](https://docs.rs/image/latest/image/struct.ImageBuffer.html#method.enumerate_pixels_mut))
  - Modify each pixel, setting it to a single colour (red) to start with ([docs](https://docs.rs/image/latest/image/struct.Rgb.html))
- Save the buffer to a file ([docs](https://docs.rs/image/latest/image/struct.ImageBuffer.html#method.save))

Your image should be saved to disk, and look like this:

![](./img/1-1.png)

### Task 1.2: Gradients

We're gonna extend the code from above to output something a bit nicer. From here on out, I'm going to talk about RGB values as being floats from 0.0 to 1.0. Converting them to bytes can be done just before you write to the pixel. I'm also going to refer to `i` and `j` as the coordinates of each pixel, where `i` is the offset in columns from the top-left corner, and `j` is the offset in rows (if you're not already using an iterator that does this for you, try and find it in the `image` documentation).

For each pixel:

- Scale `i` to the range 0.0-1.0 based upon the image's width. This will be your `r` value.
- Scale `j` to the range 0.0-1.0 based upon the image's height. This will be your `g` value.
- Fix `b` at 0.25
- Convert your rgb values to bytes
- Write your pixel to the buffer.

Red will fade from 0 to 1 left to right, and green will fade in from top to bottom. You should get a nice gradient like this:

![](./img/1-2.png)

This is a sort of graphics "Hello World", because once we have an image we can do what we want with it.

## 2: Vectors

Almost all graphics programs have some data structures for storing geometric vectors and colours. In many systems these vectors are 4D (3D plus a homogeneous coordinate for geometry, and RGB plus an alpha transparency channel for colours). For our purposes, three coordinates work just fine. We’ll use the same struct `Vec3` for colours, locations, directions, offsets, whatever. Some people don’t like this because it doesn’t prevent you from doing something silly, like adding a colour to a location. They have a good point, and we could enforce this through Rust's type system, but we're going to not for now because it adds a lot of complexity. We will create some type aliases `Colour` and `Point`, though, to make our types a little more descriptive where possible.

### Task 2.1: Vector Struct

Our `Vec3` will require a few methods to make it useful in graphics applications:

- [Dot](https://www.mathsisfun.com/algebra/vectors-dot-product.html) and [cross](https://www.mathsisfun.com/algebra/vectors-cross-product.html) products
- A `len()` method, to get its magnitude
- A `normalise()` method, to scale a vector to unity magnitude.
- A `map()` method, that applies a function to each element of the vector, consuming it and returning a new vector, similar to the `map()` method for arrays.

Create a new `vector.rs` file, and include it in the module tree with a `mod vector;` statement in main. Then create a simple struct, `Vec3`, with three `f64` fields: x, y, z. Then, implement all these methods on it. Start with `dot()` and `len()`, then try `cross()`. Do `map()` next, as you can use it to then implement `normalise()`. Look at the docs for [std::array::map](https://doc.rust-lang.org/std/primitive.array.html#method.map) for help with your map implementation, you want to take some function as an argument, and apply it to all 3 elements in your vector.

Add two type aliases `pub type Colour = Vec3` and `pub type Point = Vec3` too, you can add any other general vector methods you think might come in handy too.

Since we are using `Vec3` for colours, it's also useful to add a method to convert `Vec3` into `image::Rgb`. Rust has a pair of traits to convert [`From`](https://doc.rust-lang.org/std/convert/trait.From.html) a type and [`Into`](https://doc.rust-lang.org/std/convert/trait.Into.html) a type. These traits are the inverse of each other, so when you implement one, you get the other for free. The conventional way to do this is to implement `From` and let the compiler to `Into` for you, so go ahead and implement the trait `From<Vec3> for Rgb<u8>` and Rust will ensure `vec.into()` can convert to `Rgb` (the resulting type of `.into()` is inferred).

### Task 2.2: Vector Operations

We'll also want to overload some operators. Operator overloading allows operators to be implemented to work on custom types, which is done in Rust by implementing the [`std::ops`](https://doc.rust-lang.org/std/ops/index.html) traits. You want to be able to:

- Add two vectors
- Subtract two vectors
- Multiply a vector with a float
- Divide a vector by a float
- Negate a vector
- Multiply a vector element-wise by another vector

Implementing all of these means a lot of boilerplate, but we can draft in another crate to help us: [`derive_more`](https://crates.io/crates/derive_more), which extends the `#[derive(...)]` macros we're familiar with by allowing us to derive more traits, including operators. Add it to your `Cargo.toml` and have a look at the docs to see how to use it. Add derives for `Add`, `Sub`, `Mul`, `Div`, and `Neg`. You can also derive a `Constructor`! Add `Debug`, `PartialEq`, and `PartialOrd` while you're at it too.

Our vector is only 24 bytes, so can be considered cheap enough to derive `Copy` and `Clone` for it too. Remember that this disregards move semantics for the vector to let the compiler automatically make copies of it where needed.

`derive_more` isn't perfect, so we need to add a few operator overloads manually. `Mul<f64> for Vec3` is derived for us, which gives us `mul(Vec3, f64)`, but not the other way round (Rust does not assume that multiplication is commutative when implementing these traits). We can get the other way round with an `impl Mul<Vec3> for f64`, so we technically implement the trait again for the `f64` type. Take a look at the docs for [`std::ops::Mul`](https://doc.rust-lang.org/std/ops/trait.Mul.html) to work out how to do this.

There's also one or two cases where we want to multiply a vector by another vector element-wise. Add another `Mul` implementation for `Vec3` to do this.

### Task 2.3: Macros

We're gonna take a quick adventure down the rabbit hole that is Rust macros to create a ~~dirty hack~~ shorthand for initialising new vectors, since we're going to be doing an awful lot of it. I recommend having a read through [this blog post](https://blog.logrocket.com/macros-in-rust-a-tutorial-with-examples/), and some of the [Rust by Example](https://doc.rust-lang.org/rust-by-example/macros.html) chapter, then I'll walk you through it.

Declarative macros are just functions that operate on syntax. Our macro is declared using `macro_rules!`, and we'll call it `v!` (because it's for vectors).

```rust, noplayground
macro_rules v! {
    //patterns
}
```

The arguments are declared using syntax similar to `match`: `() => {}`. The macro matches on the pattern in the parentheses, and then expands to the code in the braces. In the parentheses goes the arguments to the macro, which are Rust syntax items, specified like `\$x: ty`, where `\$x` is the name of the token, and `ty` is the type of the syntax token. There's a few kinds of tokens, but we'll just use `expr` for now, which matches any expression, so takes almost anything.

```rust, noplayground
macro_rules v! {
    (\$x: expr) => {
        Vec3::new(\$x, \$x, \$x)
    }
}
```

The macro above takes a single expression as an argument, and replaces it with a call to `Vec3::new` with the same expression as all 3 arguments. A call to `v!(1)` will be expanded to `Vec3::new(1, 1, 1)`. We don't have to just use numbers though, the macro can be called on _any valid expression_.

We're going to add another pattern too to create a vector with three different arguments. The macro will pattern match on the two sets of arguments, and expand the one that matches. If no patterns match, the code won't compile.

```rust
macro_rules! v {
    (\$x: expr, \$y: expr, \$z: expr) => {
        Vec3::new(\$x, \$y, \$z)
    };
    (\$x: expr) => {
        Vec3::new(\$x, \$x, \$x)
    };
}
```

We'll add another neat little trick using `From` again. `f64::from` accepts any value that can be easily converted to a f64, and converts it. For example, we can do `f64::from(0_u8)`, `f64::from(0_i32)` and `f64::from(0.0_f32)`, and get `0.0_f64` from all of them. Using this in our macro lets it be a little more flexible.

```rust
#[macro_export]
macro_rules! v {
    (\$x:expr, \$y: expr, \$z: expr) => {
        Vec3::new(f64::from(\$x), f64::from(\$y), f64::from(\$z))
    };
    (\$x:expr) => {
        Vec3::new(f64::from(\$x), f64::from(\$x), f64::from(\$x))
    };
}
```

The `#[macro_export]` annotation at the top tells Rust to export our macro at the crate root, so other modules in our crate can use it with `use crate::v`. Exporting/using macros is a bit funky in Rust, but don't worry about it too much for now.

## 3: Rays

All ray tracers need some data type to represent rays. Think of a ray of a function $\mathbf P(t) = p_0 + t v$.

- $\mathbf P$ is a position on a line in 3 dimensions
- $p_0$ is the ray origin
- $v$ is the direction the ray is pointing

Using this, you can plug in a different parameter `t` to get a position anywhere on the line/ray.

{{#include ./img/rt/fig-1.02-lerp.svg}}

### 3.1: Ray Struct

Create a new `ray` module. Create a new struct in it that stores the origin `Point` and direction `Vec3` of a ray, and add a method `Ray::at(&self, t: f64) -> Point` that returns the point in 3d space that is `t` units along the ray. Either create or derive a constructor for your `Ray` too.

### 3.2: Ray Directions

Now we have rays, we can finally trace some. The basic concept is that the ray tracer casts rays from a "camera" and through each pixel, calculating the colour of each pixel. This is like tracing the light paths backwards. We'll start with a simple camera defined with a few basic parameters, and a `ray::colour` function that will trace and compute the resulting colour for a ray.

Our basic image will use the very standard 16:9 aspect ratio, partly because with a square image it's easy to introduce bugs by accidentally transposing `x` and `y`.

The camera will be at $(0, 0, 0)$, with the `y` axis going up and `x` to the left, as you'd expect. To maintain a right-handed coordinate system, the camera will face in the `-z` direction.

We'll also set up a virtual viewport that represents the screen in the world. For each pixel, we will trace a ray out into the scene. This viewport will be two units wide and one unit away from the camera (facing -z). We will traverse the screen from the upper left-hand corner, and use two offset vectors $\mathbf u$ and $\mathbf v$ along the screen sides to move the ray across the screen.

{{#include ./img/rt/fig-1.03-cam-geom.svg}}

{{#include ./img/rt/fig-1.03-cam-geom-2.svg}}

- Define your aspect ratio as `16/9`, your width as 400, and your height accordingly.
- The viewport height should be 2, and width should be set accordingly as per the aspect ratio.
- The focal length should be 1
- Looking at the diagram above, we can see that the top left corner lies at $\mathbf O - \mathbf x /2 + \mathbf y/2 - \mathbf f$
  - $\mathbf x$ and $\mathbf y$ are your image height and width vectors
  - $\mathbf f$ is your focal length vector
  - It's tempting to not bother with the vectors here, but it will become very helpful when the camera is moveable.

Write a `colour(&Ray) -> Colour` function that for now always returns `v!(0, 1.0, 0)`, we'll add a nice pattern later. Update your loop in your `main` function to calculate the direction vector of the ray to cast on each iteration based on `i` and `j`, and then create a ray starting at the origin and going into the pixel. You can do this by scaling your pixel coordinate from 0 to 1, and then multiplying by your height and width vectors. Colour your ray and save the pixel value to the RGB buffer calling `Vec3::into` to convert your colour from 0-1 from 0-255. Take care to get your signs right here, ensuring your vectors are all going in the same direction.

You should get a nice green rectangle. I appreciate there's a lot going on here and it doesn't look like much right now, so ask for help or take a look at the solutions if you're not sure.

### 3.3: Sky

To make the background for our raytraced image, we're gonna add a nice blue-white gradient. In your colour function, add code to normalise the ray's direction vector, then scale it from $-1 \leq t \leq 1$ to $0 \leq t \leq 1$. We're then gonna do a neat graphics trick called a lerp, or linear interpolation, where we blend two colours: `blended_value = (1-t) * start_value + t * end_value`. Use white for your starting colour, a nice `(0.5, 0.7, 1.0)` blue for your end colour, and blend based upon the y coordinate. You should end up with something like:

![](./img/3-3.png)

If your colours don't look similar, or it's upside down, check your geometry is correct.

## 4: Spheres

Spheres are often used in raytracers because it's fairly easy to work out if a ray has hit one or not. The equation for a sphere centred at the origin with radius $r$ is:

$$
x^2 + y^2 + z^2 = r^2
$$

This means that for any given point $(x, y, z)$, the equation will be satisfied if the point is distance $r$ from the origin. If the sphere is at centre $(C_x, C_y, C_z)$, then the equation gets a little uglier:

$$
(x - C_x)^2 + (y - C_y)^2 + (z - C_z)^2 = r^2
$$

We need to get our equation in terms of vectors instead of individual coordinates to work with them in a graphics context. Using $\mathbf P = (x, y, z)$ and $\mathbf C = (C_x, C_y, C_z)$, the vector from $\mathbf C$ to $\mathbf R$ is $(\mathbf P - \mathbf C)$, so the equation of our sphere in vector form is:

$$
|\mathbf P(t) - \mathbf C|^2 = (\mathbf P - \mathbf C) \cdot (\mathbf P - \mathbf C) = r^2
$$

We want to know if our ray $\mathbf P(t)$ ever hits the sphere. We're looking for some $t$ where the following is true:

$$
(\mathbf P(t) - \mathbf C) \cdot (\mathbf P(t) - \mathbf C) = r^2
$$

$$
(\mathbf p_0 + t \mathbf v - \mathbf C) \cdot (\mathbf p_0 + t \mathbf v - \mathbf C) = r^2
$$

A little vector algebra and we have:

$$
t^2 \mathbf v \cdot \mathbf v + 2t \mathbf v \cdot (\mathbf p_0 - \mathbf C) + (\mathbf p_0 - \mathbf C) \cdot (\mathbf p_0 - \mathbf C) - r^2 = 0
$$

The only unknown in that equation is $t$, so we have a quadratic. We can use everyone's favourite, the quadratic formula, to find a solution.

$$
t = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a}
$$

where

$$
a = \mathbf v \cdot \mathbf v
$$

$$
b = 2 \mathbf v \cdot (\mathbf p_0 - \mathbf C)
$$

$$
c = (\mathbf p_0 - \mathbf C) \cdot (\mathbf p_0 - \mathbf C) - r^2
$$

There are three possible cases, which the determinant of the formula (the $b^2 - 4ac$ bit), will tell us:

{{#include ./img/rt/fig-1.04-ray-sphere.svg}}

Empowered with some A-level linear algebra, we can go forth and draw balls.

### 4.1: Sphere Struct

Create another file `object.rs` that will contain code to do with objects. In there, create a new struct `Sphere` that holds the centre point and radius. Derive a constructor for it. Implement a method `hit` that takes a ray as an argument, and returns `true` if there is at least one intersection, and false otherwise.

Add a call to `Sphere::hit` in your `ray::colour` function, checking for intersection with a sphere with radius `0.5` centred on `(0, 0, -1)`. If there is a hit, return red instead of our usually lovely lerp background from earlier. The result:

![](./img/4-1.png)

You have a basic ray tracer that can calculate intersections, congrats! This has zero bells and/or whistles so far, but we'll get to shading and reflection later on.

### 4.2: Rayon (multi-threading)

How long did that take to execute on your machine? You might notice the raytracer starting to chug from here on out, because it's doing a lot of maths, and it'll start to do a lot more maths as we add more code. This is really what GPUs are for, but that's a whole other rabbit hole. We can do a few things to increase performance though. Introducing, my favourite crate: [`Rayon`](https://crates.io/crates/rayon).

Rayon is a data parallelism library that works by converting your iterators to parallel iterators, and then distributing work across all the cores in your system. We're going to re-write our main rendering loop as an iterator, and then drop Rayon in to make it a lot faster (depending on how many cores you have).

Where we are using for loops, we generally convert them to iterators using the [`for_each`](https://doc.rust-lang.org/stable/std/iter/trait.Iterator.html#method.for_each) adaptor, which just calls a closure on each item of an iterator. The two examples below are equivalent.

```rust
for x in 0..10_u32 {
    //loop body
    let y = f64::sqrt(x.into());
    println!("sqrt(x) = {y}");
}
```

to:

```rust
(0..10_u32).for_each(|x| {
    //the same loop body
    let y = f64::sqrt(x.into());
    println!("sqrt(x) = {y}");
})
```

Convert your rendering loop in `main` to use `for_each`. Then, import Rayon's prelude with `use rayon::prelude::*`, and add a call to [`par_bridge`](https://docs.rs/rayon/latest/rayon/iter/trait.ParallelBridge.html#tymethod.par_bridge) before your `for_each` to use Rayon's parallel bridge to parallelise your iterator. Run your code to make sure nothing is broken, and you should notice a speed-up.

Another easy way to get free speed is to run in release mode. Instead of just `cargo run`, doing `cargo run --release` will compile your code with optimisations and no debug symbols to help speed it up, at the cost of longer compile times.

There are more efficient ways to utilise rayon than this (notably `par_bridge` is not as performant as regular parallel iterators), and additional optimisations that can be enabled in `rustc`. I encourage you to play around with it and experiment to see what makes the renderer fastest.

## 5: Surface Normals & Multiple Objects

### Task 5.1: Normal Shading

A surface normal is a vector that is perpendicular to the surface of an object. You, stood up, are a surface normal to the planet earth. To be able to shade our sphere, we need to know the surface normal at the point where the ray intersects with the sphere.

{{#include ./img/rt/fig-1.05-sphere-normal.svg}}

We'll normalise our normal vectors (make them unit vectors), then visualise them using a colour map, scaling from $-1 \leq x \leq 1$ to $0 \leq x \leq 1$ like before. This means we need to do two things.

First, change your hit function to return the solution to the quadratic equation if the ray and sphere intersect, and return nothing if the ray misses. If there are two solutions, you can pick which solution is closest without comparing them directly, but I'll let you figure that out ;P.

Next, re-write your colour function to do the following:

- Check if the ray and sphere intersect
  - If they do, use the `Ray::at()` function from earlier to find the exact point $P$ where, and then find the surface normal using $\mathbf P - \mathbf C$
    - Normalise the surface normal and scale it to the range $0 \leq x \leq 1$
    - Return this as a colour to shade your sphere
  - If they do not, then just return then same background colour as before

You should get this lovely image of a shaded sphere:

![](./img/5-1.png)

### Task 5.2: Hit

Only one sphere is boring, let's have some more. Let's create a trait to represent objects so we can easily extend our raytracer with whatever we want. The `Object` trait will contain our `hit()` function, so any shape/object/thing can then implement it to be able to tell us if a ray has hit it or not.

We'll extend the `hit()` function a bit here to, to be something more like `fn hit(&self, ray: &Ray, bounds: (f64, f64)) -> Option<Hit>`. The `bounds` will specify valid bounds for the parameter `t` (the solution of our quadratic equation) to lie in, and
the `Hit` struct will bundle some more detailed information about a ray-object intersection. `Hit` will include:

- The `Point` where the intersection is
- The surface normal
- The parameter `t`

Create the `Hit` struct, add the `Object` trait with its one function, and then implement it for `Sphere`. The old `Sphere::hit()` can go, as the new and improved `Sphere::hit()` should be part of the `Object` impl. You still need to determine if there is an intersection or not using the same calculations as before, and you'll now need to do some additional maths to find the closest of the two roots that is in within the bounds given. Calculate the surface normal and the intersection point here too, and pack it all into the `Hit` struct. If there is no intersection, continue to return `None`.

Update your colour function to use the `Object` trait implementation of `Sphere::hit`. Put the bounds as $0 < x < \infty$ for now, so all intersections in front of the camera are valid. Also update it to deal with the new return type, but doing the same thing as before, shading based upon the normal. Make sure there's no change from the last task so you know everything is working.

### Task 5.3: Inside Out

We need to talk about surface normals again. The normal is just a unit vector perpendicular to the surface, of which there is actually two: either pointing outside the sphere, and the inverse pointing inside. At the moment, the normal we're finding will always match whether the ray starts inside or outside the sphere (we will need inside when we get to glass). We need to know which side a ray hits from, and we'll need to be consistent in which direction the normals point.

{{#include ./img/rt/fig-1.06-normal-sides.svg}}

For consistency, we're going to make normals always point outward, and then store in the `Hit` struct which side of the object the ray hit. Add a boolean field called `front_face` to the struct, that will be `true` when the ray hits the outer surface, and `false` when it hits the inner surface.

The normalised outer surface normal can be calculated by $\frac{1}{r} (\mathbf v_0- \mathbf C)$ (the length of $\mathbf v_0- \mathbf C$ is $r$, so this normalises). We can then use a property of the dot product to detect which side the ray hit from:

$$
\frac{\mathbf x \cdot \mathbf y}{ |\mathbf x||\mathbf y| } = \cos \theta
$$

Where $\theta$ is the angle between the two vectors joined tip-to-tip. This means that if the dot product of two vectors is 0, they are perpendicular. If the product is positive the two are at angles of less than 90 degrees, and if negative they lie at angles of between 90 and 180 degrees. So, if `ray.direction.dot(outward_normal) > 0`, then the ray has hit from the inside and we need to invert our normal and set `front_face` to false.

Implement this logic in your code, making sure that in the current render `front_face` is always true. If there are any bugs in your implementation you might not catch them all now because we have no cases where `front_face` is false yet, so double and triple check your maths. You could shuffle the sphere and camera positions around to put the camera _inside_ the sphere, and see what your results look like.

### Task 5.4: Scenes

We have implemented `Object` for `Sphere`, but what about multiple spheres? We don't want to check _every_ object for intersection for each ray, so we're going to implement `Object` for a list of `Objects`. Well how is that going to work, can you implement a trait for a list of itself? Yes, you can. We're going to use [trait objects](https://doc.rust-lang.org/book/ch17-02-trait-objects.html). Trait objects are pointers to types, where the only thing known about that type is that it implements a specific trait. If you're familiar with dynamic dispatch, this is dynamic dispatch in Rust. We can create a `Vec<dyn Object>`, which Rust reads as "a list of items which all implement `Object`, and I'll figure the rest out at runtime".

Create a type alias for an entire scene: `pub type Scene = Vec<dyn Object>`. The compiler should now be complaining, as it can't know what the size of `dyn Object` is at compile time, so we can't just put it into a `Vec` so easily. We need to put all our types in **boxes**, which are smart pointers that work with data of unknown size (allocated on the heap, not on the stack). If you haven't figured it out already, what you actually want is `Vec<Box<dyn Object>>`.

Now we have the type nailed down, implement `Object for Scene`. The idea is you return the `Hit` that is closest to the camera, so the one with the smallest `t` of all the spheres. Being able to do this provides some nice abstraction, as we can just check if the ray intersects anywhere in the scene, and get back the `Hit` for the closest object, which is the one the ray actually wants to hit.

The code you wrote using Rayon earlier might be complaining now. Rust is very strict about what types can work with multithreaded code, which it enforces though the [Send and Sync traits](https://doc.rust-lang.org/book/ch16-04-extensible-concurrency-sync-and-send.html). Rust doesn't know for sure that a `dyn Object` is necessarily safe to share between threads, so we simply need to add a `Sync` bound on our `Scene` type signature: `Vec<Box<dyn Object + Sync>>`. All our types implementing `Object` will are (and will be) `Sync`, so we don't need to worry about this, beyond us telling the compiler and Rayon know we have everything under control. Change your type signature again to be `Vec<Box<dyn Object + Sync>>`, and that should shut the compiler up.

### Task 5.5: "Ground"

In main, create a `Scene` of the existing sphere, then add another sphere at `(0, -100.5, -1)` with radius 100. Change your colour function to take anything that implements `Object` (it's probably best to use generics here), and you should end up with something like this:

![](./img/5-5.png)

It's quite convenient the "ground" appears green, since that happens to be the colour for a normal with high `y` (`G`) and small `x` and `z` (`r` and `b`). Note how it occurs on the top of the small sphere too.

## 6: Antialiasing

You might have noticed the slightly jagged and pixelated edges on your sphere. This is because we are taking only a single ray (sample) per pixel, which is either sphere or background -- when many pixels will cover a region that is partly both. To combat this, we will use a form of anti-aliasing called supersampling.

{{#include ./img/rt/fig-antialias.svg}}

Supersampling is simply using multiple rays (samples) per pixel, which are averaged to form the final colour value. We'll be using one of the simplest forms: averaging the colour of a set of randomised samples (rays) per pixel.

### Task 6.1: Multisampling

So that each sample isn't identical, we're going to slightly randomise the ray directions within each pixel, for which we will use our old friend, the `rand` crate. Add it to your package manifest. `rand::random::<T>()` is a generic function that will return a random `T`. In case of floats, `rand::random::<f64>()` returns a random number $0 \leq x < 1$.

In your main rendering loop, add a slight randomness to your ray direction by adding a random float to your _pixel_ coordinate. Add this before you scale the coords down to the viewport (dividing by image size). This detail is important otherwise you'll be casting rays all over the place and your render will be totally screwed.

Add a variable `let samples = n` to the top of `main`. Update the render loop to take `n` random samples of each pixel, and then set the final pixel colour to the average of the `n` samples. Your render should be the same, but with smoother edges to your sphere's edges. Zoom in around the edge of the sphere and you should notice the effect. Use `n=100` samples for now, but don't hardcode this in your loop.

![](./img/6-1.png)

### Task 6.2: Camera Struct

Now seems like as good a time as any to abstract the camera out into its own type. Create a new file `camera.rs`, and in it add a `Camera` struct, that contains:

- The camera's position, the ray origin
- The position of the top left corner of the viewport
- The horizontal and vertical viewport size vectors.

Add a function to return the default camera, with the exact same configuration as before. Add also a method `get_ray()` and move in the code that takes the coordinates of a pixel and returns the ray from the camera through it.

Check you haven't introduced any bugs by making sure your render is the same as before.

### Task 6.3: Progress Bars

Taking 100 samples for each pixel is probably making your renderer start to chug again. If it's really taking too long, try dropping the number of samples, but we can add a progress bar as a nice little touch to help us see how long it's got left. We're going to use another crate: [`indicatif`](https://crates.io/crates/indicatif/0.16.2) (0.16 is required for the below styling to work, 0.17 changes the syntax).

Indicatif works by binding a progress bar to iterators, and then shows a progress bar in the console as the iterator progresses. Have a read over the docs and examples to get an example of how it works.

Remember that we're using Rayon's parallel iterators instead of regular iterators? Indicatif has support for those too. In Rust, crates can add optional features that are enabled/disabled in the package manifest. By enabling Indicatif's `rayon` feature, it will work correctly with parallel iterators. Add the following line to your dependencies:

```toml
[dependencies]
indicatif = { version = "0.16", features = ["rayon"] }
```

Add a progress bar with a given length to your program by declaring one in main using `ProgressBar::new()`. Configure its style and format to your liking (the style I used is shown below). Add it to your parallel iterator using `progress_with()`.

```rust, noplayground
bar.set_style(
    ProgressStyle::default_bar()
        .template(
            "{spinner:.green} [{wide_bar:.green/white}] {percent}% - {elapsed_precise} elapsed {msg}",
        )
        .progress_chars("#>-")
        .on_finish(ProgressFinish::WithMessage("-- Done!".into())),
);
```

By default, `indicatif` updates and redraws the progress bar for every update, however we have hundreds of thousands of updates, so this can add significant lag. Limit this draw rate to $x$ times a second with:

```rust, noplayground
bar.set_draw_rate(5);
```

![](./img/progress-bar.png)

Indicatif lets you create some really neat progress bars, so have a play around with it to customise it to your liking.

## 7: Diffuse Materials

We're about ready to start making objects look realistic. Diffuse objects (that don’t emit light) appear as a mix of their own colour spectrum and the colour of the light reflected off their surroundings. Diffuse objects randomly scatter the incoming light rays as they have a rough surface (at least at a microscopic level). So, if we send three rays into a crack between two diffuse surfaces they will each have different random behaviour, as shown.

{{#include ./img/rt/fig-1.08-light-bounce.svg}}

Light rays may also be absorbed rather than reflected. Darker surface have higher levels of absorption (that's why it's dark, they don't reflect light). [Lambertian reflectance](https://en.wikipedia.org/wiki/Lambertian_reflectance) is the property that defines an ideal diffusely reflecting surface, and we're going to model it.

Light hitting the surface at an angle further from the normal have less colour influence, as they hit at a shallower angle. This property can be closely modelled by sampling in a unit sphere which has the surface as a tangent.

The unit radius sphere tangent to the hit point $p$ of a surface which is outside the surface. This has its centre at $(\mathbf P + \mathbf n)$, where $\mathbf n$ is the normal to the surface at $\mathbf P$. Pick a random point $S$ inside the unit radius sphere and send a ray from the hit point $\mathbf P$ to the random point $\mathbf S$, to give us a random vector $(\mathbf S - \mathbf P)$, that will be the diffuse reflected ray.

{{#include ./img/rt/fig-1.09-rand-vec.svg}}

We need a way to pick a random point in a unit sphere. A rejection method is the easiest way to do this: pick a random point in a unit _cube_, and reject it and try again if it's not in the sphere. If we then normalise the vector to be actually _on the sphere_, then it actually more accurately models Lambertian reflection. The original book has a [much more detailed discussion of modelling diffuse reflection](https://raytracing.github.io/books/RayTracingInOneWeekend.html#diffusematerials/truelambertianreflection), and I encourage you to read over it.

### Task 7.1: Lambertian Sampling

Write a function to do this by generating a vector whose elements are random numbers between -1 and 1. If the length of the vector is less than 1, then it's in the sphere.

We're going to update the colour function to be recursive, casting reflection rays in a random direction. The direction of the reflected ray will be $\mathbf n + \mathbf S$, and will come from the impact point of the original ray (you can get all that data from your `Hit` struct, remember). Halve the result of the recursive call so that each reflected ray has less and less intensity, and return that.

### Task 7.2: Recursion Limit

If you ran this, you probably blew the stack with infinite recursive calls. We need to limit the number of child rays to prevent infinite recursion. Add a depth parameter to the `colour` function to keep track of the current recursion depth. When the recursion hits max depth, say 50 rays for now, stop recursing and just return black.

You should have an image like so

![](./img/7-2.png)

### Task 7.3: Gamma Rays

Take a look at the shadowing under the sphere. This picture is very dark, but our spheres only absorb half the energy on each bounce, so they are 50% reflectors. If you can’t see the shadow, don’t worry, we will fix that now. These spheres should look pretty light, about a light grey. The reason for this is that almost all image viewers assume that the image is “gamma corrected”, meaning the 0 to 1 values have some transform before being stored as a byte. There are many good reasons for that, but for our purposes we just need to be aware of it. To a first approximation, we can use “gamma 2” which means raising the colour to the power 1/gamma, or in our simple case 0.5, which is a square root.

Add a line to the `From<Vec3> for Rgb<u8>` function to correct for this, taking the square root of all the values before converting them to bytes. Your render should look a bit lighter now, closer to what you'd expect for a 50% reflector:

![](./img/7-3.png)

### Task 7.4: Shadow Acne

There is another subtle bug here. Some of the reflected rays aren't reflecting from exactly $t=0$, but a small fraction off it, do to floating point imprecision. This can lead to some rays starting slightly behind their starting surface, erroneously then re-intersecting with it. Ignoring hits very near to 0 can fix this, by passing the minimum bound as `0.00001` instead of `0` to the hit function. The updated render looks lighter and much cleaner:

![](./img/7-4.png)

This fixes the problem known as "shadow acne".

## 8: Metal

We have diffuse materials modelled by Lambertian reflectance, so let's add a metal material. Before we do that, we'll create some abstractions for dealing with different materials and reflections.

### Task 8.1: Lambertian Material

We're going to create a trait to describe how light scatters off different materials:

```rust, noplayground
pub trait Material {
    fn scatter(&self, incident_ray: &Ray, hit: &Hit) -> Option<Reflection>;
}
```

`Material::scatter` takes the incident ray and the `Hit` struct, and uses them to determine if there is a reflected ray. The `Reflection` struct contains the reflected ray, and also a vector to describe how the colour of the reflected ray is attenuated (because different materials will change the colour of reflected rays differently). Create a new file `material.rs`, and add `Material` and `Reflection` there.

We'll add an implementation of `Material` to describe Lambertian reflectance. The `Lambertian` struct needs a single field to describe how the colour of reflected rays is attenuated.

The `Material` impl for `Lambertian` should contain the logic that's currently in `ray::colour`. Calculate the scatter direction, create the reflected ray, and return it inside a `Reflection` struct. The amount the reflected ray is attenuated by is the colour of the material.

### Task 8.2: Use Materials

We need to make our objects aware of the fact that they can be different materials too. The `Sphere` struct needs an extra field, `material`, the type of which should be any type implementing the `Material` trait. That's right, your struct is going to need to be generic.

Add a field `reflection` to the `Hit` struct too. The idea is that then the `Object::hit` method populates that field if the hit caused a reflected ray to be generated. Update `Sphere::hit` to call the `Material::scatter` from it's `material` field, and use that to fill the `reflection` field of `Hit`.

Update `ray::colour` to use the new `Material` abstraction. If there is a reflected ray, make the recursive call same as before, returning the result multiplied by the colour attenuation.
You'll need to update the two spheres created in `main` too to account for this. Make them both have colour `(0.5, 0.5, 0.5)`

There's a lot of re-architecting of the raytracer going on here, but nothing actually changes functionally yet. Make sure the rendered image is the same. If you set the random seed to be the same, Git will even tell you if your file has changed or not!

### Task 8.3: Edge Cases

Take another closer look at `Lambertian::scatter`. If the random unit vector generated is _exactly_ opposite the normal vector, the two will sum to zero and the scatter direction will be zero, meaning we'll have no reflected ray. In our implementation, this means we get a black pixel where we should have colour. We need to avoid this by checking if the scatter direction is zero, and if it is we'll just reset the scatter direction to the normal.

Floating point zero is weird because it isn't exact (we already had to deal with shadow acne), so add a method to your vector to check if it is zero, returning `true` if all three of its elements are within some small tolerance of zero (e.g. `1e-8`). If this is the case, replace the scatter direction with the hit normal.

### Task 8.4: Smooth Reflection

Now we've built up our abstractions and tested they work on the existing logic, we can extend the raytracer to include a metal material.

For smooth metal the ray won't be randomly scattered. We need to do the maths to work out the direction of reflected rays.

{{#include ./img/rt/fig-1.11-reflection.svg}}

The reflected ray direction in red is $\mathbf v + 2 \mathbf B$. The length of $\mathbf B$ is $\mathbf v \cdot n$ and it goes in the same direction as $\mathbf n$, so we can calculate it as $(\mathbf v \cdot \mathbf n) \mathbf n$. So we have:

$$r = v - 2 (\mathbf v \cdot \mathbf n) \mathbf n$$

(Note the minus sign, since $v$ points inwards and $r$ outward.)

Write a function `reflect(v: Vec3, normal: &Vec3) -> Vec3` to implement this. You could put this as a private method of `Metal`, or as a module-scope function at the bottom of `material.rs`.

This function is used to reflect rays off our metal material. Add a struct `Metal` that has a single `Colour` field, and implement `Material` for it, calculating this reflected ray. Note that the reflected ray should only be returned if the dot product of it's direction with the hit normal is greater than zero. A reflection ray that is at an angle of greater than 90 degrees to the normal doesn't make sense (how are you reflecting from under the surface?).

Update your scene so you have four spheres:

- Center `(0, 0, -1)`, radius `0.5`, `(0.7, 0.3, 0.3)` lambertian
- Center `(-1, 0 -1)`, radius `0.5`, `(0.8, 0.8, 0.8)` metal
- Center `(1, 0, -1)`, radius `0.5`, `(0.8, 0.6, 0.2)` metal
- Center `(0, -100.5, -1)`, radius `100`, `(0.8, 0.8, 0)` lambertian

Your new render should look like this. See how the metal spheres are reflecting the centre sphere, and you can see the other half of the ground sphere behind them.

![](./img/8-4.png)

### Task 8.5: Fuzziness

Right now the metals look perfectly smooth, which is very rarely the case in real life. We can add a little fuzziness by randomising the reflected rays slightly, with a fuzz factor. We'll change the scattered ray direction to be $\mathbf r + f \mathbf S$, where `f` is a scalar fuzz factor, and $S$ is a random unit vector (which you hopefully have from earlier). Add a `fuzz` field to the `Metal` material struct, and update the reflected ray direction to be a little fuzzy, reusing the random unit vector function from earlier.

Make the left sphere have a fuzziness of `0.3`, and the right sphere `1.0`. Your updated render should look like this:

![](./img/8-5.png)

## 9: Glass

Clear materials such as water and glass are [dielectrics](https://en.wikipedia.org/wiki/Dielectric). When light hits them, it splits into a reflected ray and a refracted ray. We'll model this in our raytracer by randomly choosing between either reflection or refraction, and only generating one or the other per sample.

Refraction is described by Snell's law:

$$
\eta \sin \theta = \eta ' \sin (\theta ')
$$

where $\theta$ and $\theta '$ are angles from the normal, and $\eta$ and $\eta '$ are the refractive indices of the two materials. We want to solve for $\theta '$, to get the angle of our new ray.

{{#include ./img/rt/fig-1.13-refraction.svg}}

On the refracted side of the surface there is a refracted ray $\mathbf{R} '$ and a normal $\mathbf{n}'$, and an angle $\theta '$ between them. The ray $\mathbf{R} '$ can be split into the sum of it's two components parallel and perpendicular to $\mathbf{n}'$:

$$
\mathbf{R}' = \mathbf{R}_{\perp}' + \mathbf{R}_{\parallel}'
$$

We can solve for both those components:

<details>
<summary><i>click to expand the derivation of this</i></summary>

> First we solve for $R'_\perp$. The first step here is to construct the perpendicular vector to the normal $n$.

{{#include ./img/rt/fig-1.13-refraction-deriv.svg}}

> The green vector in the diagram is parallel to $R'_\perp$, and is equivalent to the combined vectors of $R$ then back up along the normal $n$ by $\cos \theta$, hence $R + n \cos \theta$.
>
> The current magnitude of this vector is $\sin \theta$, so to normalise we divide by $\sin \theta$. The $R'_\perp$ magnitude we want is $\sin \theta'$, so multiply by that: $\frac{\sin \theta'}{\sin \theta} (\mathbf{R} + \mathbf{n}\cos\theta)$. But $\frac{\sin \theta'}{\sin \theta} = \frac{\eta}{\eta'}$, leading to our final $R_\perp' = \frac{\eta}{\eta '} (\mathbf{R} + \mathbf{n}\cos\theta)$
>
> Onto $R'_\parallel$: The total magnitude of $R'$ is $|R'| = {1} = \sqrt{|R'_\perp|^2 + |R'_\parallel|^2}$. Rearrange this for $|R'_\parallel|$ to $\sqrt{1 - |R'_\perp|^2}$, which is simpy setting the magnitude of $R'_\parallel$ to whatever makes $R'$ a unit vector. This is in the opposite direction to the normal, so use $-n$ as the direction. To simplify, $|R'_\perp|^2$ can also be expressed as a dot product of $R'_\perp$ with itself. This gives us the final $\mathbf{R}_{\parallel}' = - \sqrt{\left|1 - \mathbf{R}_{\perp}'\cdot\mathbf{R}_{\perp}' \right|} \mathbf{n}$

</details>

$$\mathbf{R}_{\perp}' = \frac{\eta}{\eta '} (\mathbf{R} + \mathbf{n}\cos\theta)$$
$$\mathbf{R}_{\parallel}' = - \sqrt{\left|1 - \mathbf{R}_{\perp}'\cdot\mathbf{R}_{\perp}' \right|} \mathbf{n}$$

Note that this still depends on knowing $\cos \theta$, but we can get this from the dot product:

$$\mathbf{a} \cdot \mathbf{b} = |\mathbf{a}||\mathbf{b}| \cos\theta$$

If we restrict $\mathbf{a}$ and $\mathbf{b}$ to be unit vectors, ie $\mathbf{a} = \mathbf{b} = 1$, then:

$$\mathbf{a} \cdot \mathbf{b} =  \cos\theta$$

This gives us $\mathbf{R}_{\perp}'$ in terms of known quantities (note $-R$ as opposite direction to $n$):

$$\mathbf{R}_{\perp}' = \frac{\eta}{\eta '} (\mathbf{R} + \mathbf{n}(-\mathbf{R} \cdot \mathbf{n})) $$

### Task 9.1: Refraction

Write a small helper function in `material.rs` to return the direction of the refracted ray. There's a lot of maths here, but the idea is:

- Take the incident ray, the normal, and the ratio $\frac{\eta}{\eta '}$ as arguments
- Calculate $\cos \theta$
- Calculate $\mathbf{R}_{\perp}'$ and $\mathbf{R}_{\parallel}'$
- Return the sum of the two

### Task 9.2: Dielectric

You can refract rays, so let's add a dielectric material that does just that with it's scatter method. Create a new struct `Dielectric` with a single field, it's refraction ratio ($\frac{\eta}{\eta '}$). Create a new `Material` impl for it, such that `scatter` returns a new reflected (\*technically it's refracted now) ray with a colour attenutation of 1 (no attenuation), and direction vector calculated by your refract function. Don't forget to normalise your incident ray before using it, as we made the assumption that $\mathbf{a} = \mathbf{b} = 1$ when we did the maths above.

An interesting thing to note is that if your ray comes from outside the sphere (ie, `hit.front_face == true`), then you will need to set the refraction ratio to be it's reciprocal, as $\eta$ and $\eta '$ are flipped.

Update the scene to change the left sphere to be a dielectric with a ratio of 1.5 (roughly glass), then render it and see what you get.

![](./img/9-2.png)

### Task 9.3: Total Internal Reflection

That _might_ not look quite right, which is because there's a flaw in our approximations.

When the ray is going from a material with a high refractive index to one with a lower one, there is no solution so Snell's law. Referring back to it:

$$
\sin\theta ' = \frac{\eta}{\eta '} \sin \theta
$$

If the ray is inside glass and outside is air ($\eta = 1.5$, $\eta ' = 1.0$), then:

$$
\sin\theta ' = \frac{1.5}{1.0} \sin \theta
$$

But the value of $\sin\theta ' $ cannot be greater than 1, so the equality is broken and the solution does not exist: the glass cannot refract. In this case the ray _must_ be reflected, which gives us the phenomenon known as total internal reflection.

Update `Dielectric::scatter` to account for this, implementing total internal reflection when it cannot refract. You'll need to calculate both $\cos\theta = \mathbf{R}\cdot\mathbf{n}$ and $\sin\theta = \sqrt{1 - \cos^2 \theta}$, then if $\frac{\eta}{\eta '}\sin\theta > 1.0$, reflect instead of refract.

You should get something that looks a bit more correct:

![](./img/9-3.png)

If you can't see much of a difference between the two for this scene, I wouldn't blame you. For the example scene, total internal reflection is never really visible as it will only happen as a ray tries to go from glass to air, so play around with the scene and see if you can see the difference.

### Task 9.4: Schlick

Let's play around with our glass model again. Real glass has a reflectivity that varies with angle (look at a window from a steep angle and it becomes more of a mirror). We're going to implement this behaviour using a neat polynomial approximation called the [Schlick approximation](https://en.wikipedia.org/wiki/Schlick%27s_approximation) (because the [actual equation](https://en.wikipedia.org/wiki/Fresnel_equations) is very big and a bit scary).

The reflectance coefficient, $R$ can be approximated by:

$$
R (\theta) = R_0 + (1-R_0)(1-\cos\theta)^5
$$

where

$$
R_0 = \left( \frac{1-n}{1+n} \right)^2
$$

$n$ is the refractive index of the material, and $\theta$ is the angle between the normal and the incident light ray. Implement a function that calculates $R(\theta)$, taking $\theta$ and $n$ as parameters.

We'll use the function by checking if $R(\theta)$ is greater than a random double every time we call `Dielectric::scatter`, and reflect instead of refract if so. You should also still be reflecting if the conditions for total internal reflection are met.

Notice how the sphere looks a little fuzzier around the edges, and a bit more realistic?

![](./img/9-4.png)

## 10: Positionable Camera

Cameras are hard, and there's a lot of geometry here so follow closely.

## 10.1: FoV

We'll start by allowing an adjustable field of view (FoV): the angle that can be seen through camera. The image isn't square so the FoV will be different horizontally and vertically. We'll specify the vertical one in degrees.

The image below shows our FoV angle $\theta$ coming from the origin and looking into the $z=-1$ plane, same as before. $h$ is therefore the height of our viewport, $h = \tan \frac{\theta}{2}$. The total height of our image will be $2h$, and the width will be `height * aspect_ratio`.

{{#include ./img/rt/fig-1.14-cam-view-geom.svg}}

Create a `new()` function for `Camera` to take an FoV and aspect ratio as arguments, and then calculate the viewport width and height instead of hardcoding them. Make sure you check your angle units!

Update your scene to use a camera with a 90 degree vertical FoV and 16:9 aspect ratio. Delete all the current objects, and add two spheres:

- We'll use a constant $R = \cos \frac{\pi}{4}$ for brevity
- Centre $(-R,0,-1)$, radius $R$, blue lambertian material
- Centre $(R,0,-1)$, radius $R$, red lambertian material

Your image should look like this, a wide-angle view of the two spheres. Play around with the FoV and see what it looks like. Making it wider will zoom out on the spheres, and narrower will zoom in. This can be a little fiddly to implement, but do make sure it's completely correct.

![](./img/10-1.png)

## 10.2: Direction

The next step is being able to move the camera to wherever we want. Consider two points: `look_from`, the position of the camera where we are looking from; and `look_at`, the point we wish to look at.

{{#include ./img/rt/fig-1.15-cam-view-dir.svg}}

This gives us the location and look direction of the camera, but does not constrain the roll, or tilt, of the camera. Think about it as if you're looking from your eyes to a point, but you can still rotate your head about your nose. We need an "up" vector for the camera, which can be any vector orthogonal to the view direction (we can actually just use any vector we want, and then project it onto the plane). This will be the "view up" vector, $v_{up}$.

We can use vector cross products to form an [orthonormal basis](https://en.wikipedia.org/wiki/Orthonormal_basis) $(u, v, w)$ to describe our camera's orientation: 3 unit vectors all at right angles to each other which describe the 3D space.

{{#include ./img/rt/fig-1.16-cam-view-up.svg}}

- $v_{up}$ is any vector that we specify
- $w$ is the vector `look_from - look_at`, so $-w$ is our view direction
- $u$ is the unit vector of the cross product $v_{up} \times w$
- $v$ is the cross product $w \times u$
  - You cannot assume $v = v_{up}$ as we allow $v_{up}$ to not be orthogonal to the view direction

Update the camera `new()` function to take `look_from` and `look_at` points as parameters, as well as a `vup` vector. Calculate `u`, `v`, and `w`, making sure to normalise them all. The new camera origin is `look_from`, and the new horizontal/vertical vectors are `u` and `v` with magnitudes of the viewport's size. The top left corner of the viewport is calculated as before, but the `-z` distance from the origin is now `w` instead of just `1.0`.

Change the scene to the following:

- Sphere centre $(0,0,-1)$, Radius 0.5, Lambertian with colour $(0.1, 0.2, 0.5)$
- Sphere centre $(-1,0,-1)$, Radius 0.5, Dielectric with ratio 1.5
- Sphere centre $(1,0,-1)$, Radius 0.5, Metal with colour $(0.8,0.6,0.2)$ and 0 fuzziness
- Sphere centre $(0,-100.5,-1)$, Radius 100, Lambertian with colour $(0.8, 0.8, 0)$
- Camera looking from $(-2, 2, 1)$ to $(0, 0, -1)$ with `vup` $(0, 1, 0)$, 90 degree FoV and 16/9 aspect ratio

Your scene, as viewed from far away, should look like this:

![](./img/10-2-1.png)

You can zoom in a bit too. Change the camera settings to a 20 degree FoV:

![](./img/10-2-2.png)

## 11: Depth of Field

Our final feature will be Depth of Field, or defocus blur. Due to camera lenses and focus, only a small range of depths will appear in focus, anything too close or far will be out of focus and blurred.
There is a plane in our image where everything will be in perfect focus, and everything closer or further will blur (only slightly at first). The distance between this focus plane and our camera lens is the focus distance. Real cameras have complex compound lenses, but we will use a thin lens approximation, more similar to how the eye works:

{{#include ./img/rt/fig-1.17-cam-lens.svg}}

The range of rays that end up on a particular point of the resulting image go through all parts of the circlar lens, and point from there towards the focus point. Any object closer (or further), will have these intersection points (that should perfectly match for perfect focus) spread over its surface, making it out of focus.

Since our camera is infinitesimally small, we'll need to pretend it has an actual lens.

{{#include ./img/rt/fig-1.17-cam-lens-2.svg}}

We'll implement this by making the origin of each ray a random point within a disc centered on the `look_from` point, to model a lens. The larger the disc (lens), the larger the defocus blur.

We'll need a simple function to generate a random point within a unit circle, so using a similar technique as we did earlier, write a function to generate a vector with random `x` and `y` components with `z = 0` that lies within a unit circle.

Update `Camera::new()` to take an aperture width (lens size) and focus distance as parameters. The horizontal and vertical image size vectors should be scaled by the focus distance, and the top left corner should be moved backwards by scaling `w` by the focus distance too. Add fields to the `Camera` struct to store `u` and `v`, as well as the lens radius (half the aperture width).

`Camera::get_ray()` needs updating to randomise the ray origins. Start by generating a random point within the lens $R$, then calculate the new ray origin as $O = \text{origin} + u R_x + v R_y$. The position of the target pixel and can then be calculated as before to generate the ray.

Update the camera to the following settings:

- `look_from` $(3, 3, 2)$
- `look_at` $(0, 0, -1)$
- `vup` $(0, 1, 0)$
- FoV 20
- Aspect ratio 16/9
- Aperture 2.0
- Focus distance as the length between `look_from` and `look_at`

The focus centre of the blue sphere lies in the focus plane so that is in focus, but the other two are blurred, as they are out of focus:

![](./img/11.png)

This specific scene looks a bit rubbish, so let's do one that's a bit nicer.

## 12: A Final Render

So our ray tracer now has everything we need to make the procedurally generated image I showed you at the very top. I'll just give you the code to generate the scene because writing out a description of how to do this would be not very practical.

```rust, noplayground
fn random_scene() -> Scene {
    let mut objects: Scene = vec![];

    let ground = Box::new(Sphere::new(
        v!(0, -1000, 0),
        1000.0,
        Lambertian::new(v!(0.5, 0.5, 0.5)),
    ));
    objects.push(ground);

    for a in -11..11 {
        for b in -11..11 {
            let a = a as f64;
            let b = b as f64;
            let material_choice: f64 = rand::random();
            let center = v!(
                a + 0.9 * rand::random::<f64>(),
                0.2,
                b + 0.9 * rand::random::<f64>()
            );

            if material_choice < 0.8 {
                //diffuse
                let material = Lambertian::new(v!(rand::random::<f64>()));
                objects.push(Box::new(Sphere::new(center, 0.2, material)));
            } else if material_choice < 0.95 {
                //metal
                let colour = v!(rand::random::<f64>() / 2.0 + 0.5);
                let fuzz = rand::random::<f64>() / 2.0;
                let material = Metal::new(colour, fuzz);
                objects.push(Box::new(Sphere::new(center, 0.2, material)));
            } else {
                //glass
                objects.push(Box::new(Sphere::new(center, 0.2, Dielectric::new(1.5))));
            }
        }
    }

    objects.push(Box::new(Sphere::new(
        v!(0, 1, 0),
        1.0,
        Dielectric::new(1.5),
    )));
    objects.push(Box::new(Sphere::new(
        v!(-4, 1, 0),
        1.0,
        Lambertian::new(v!(0.4, 0.2, 0.1)),
    )));
    objects.push(Box::new(Sphere::new(
        v!(4, 1, 0),
        1.0,
        Metal::new(v!(0.7, 0.6, 0.5), 0.0),
    )));
    objects
}
```

Use the following camera and image settings:

- Image width of 1200 pixels
- `look_from` $(13, 2, 3)$
- `look_at` $(0, 0, 0)$
- `vup` $(0, 1, 0)$
- FoV 20
- Aspect ratio 1.5
- Aperture 0.1
- Focus distance 10

![](./img/final-render-2.png)

An interesting thing you might note is the glass balls don’t really have shadows which makes them look like they are floating. This is not a bug — you don’t see glass balls much in real life, where they also look a bit strange, and indeed seem to float on cloudy days. The glass balls don't block light like a solid sphere does, it just bends the light, so plenty of light still hits the surface underneath it.

## What next?

What you've built is actually an incredibly cool thing, modelling physical light using a bit of simple geometry to build the basis of a 3D rendering engine, the principles of which are not dissimilar to that used in real gaming and 3D animation applications.

Play around with the scene, change the objects and their positions, put the camera at weird angles, see what cool pictures you can generate.

There are two more books that follow on from this [Ray Tracing: The Next Week](https://raytracing.github.io/books/RayTracingTheNextWeek.html) and [Ray Tracing: The Rest of Your Life](https://raytracing.github.io/books/RayTracingTheRestOfYourLife.html), which go on and add a bunch more features to the ray tracer. This was only up to the end of the first book so, the others are certainly worth a read, though you'll have to [carcinise](https://en.wikipedia.org/wiki/Carcinisation) it yourself (or do it in C++, which despite all it's problems is still widely used and a good skill to have).
