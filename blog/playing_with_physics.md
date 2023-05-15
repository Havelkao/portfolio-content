<!-- # Playing with physics -->

Rendering stuff with threejs is fun but on its own it gets old rather quickly. Static scenes may look nice, but if there is no interactivity, what is the point? Fortunately there is a number of ways how to spice things up. I imagine for a beginner the most obvious way would be to write a script the moves things around. If you are unfamiliar with the topic the chances are you are going to end up with a scene and an object whose position is incrementally updated in the animation loop. The scene might not be static anymore, but the movement for sure is. Here is a short example:

```js
let scene, camera, renderer, cube;

function init() {
  (...)

  cube = new THREE.Mesh(geometry, meterial);
  scene.add(cube)
}

function animate() {
  requestAnimationFrame(animate);
  cube.position.x += 0.01;
  renderer.render(scene, camera);
}
```

A slightly more sophisticated approach would be adding some sort of user input, however, the end result would technically be the same. The only difference is that the movement will stop on given condition. Say you would like to add some rotation to the object so that it doesn't only move, it rolls over. Well, with this approach you are screwed since it it rather unlikely that you will stop the input at the exact moment our cube rolls forward 90 degrees. Well is it not just another problem you can solve with a condition? Or perhaps you could use an animation library that will handle the start and end state, just like this [clueless man](https://codepen.io/Havelka/pen/OJRyedG) 2 years ago. Apparently this is the solution, word by word ^^

`"I’m rotating and animating the pivot around the respective axis, adjusting the position and rotation of the cube and finally resetting the the rotation of the pivot."`

Anyways, it is not too difficult to imagine why this is not exactly the way to go about this issue. I may or may not have written the quotation above and after reading it a couple of times I still have no idea what the hell is going on, or what would I do if I wanted to extend the functionality.

## Enter physics engine

A much better solution is using a physics engine. With such engine not only would the desired rotation look more natural, you would also have to spend a lot less effort to achieve the same result. No need to reinvent the wheel, however

Technically speaking it is a tredeoff between trying to reinvent the wheel and learning how to integrate an existing solution. In this case I'd much rather do the latter.

There are multiple available engines to be used, each of which appear to have different learning curve. By far the most popular is cannon-es a maintained version of now deprecated cannonjs. So how does it work? With a physics engine you are in fact not directly manipulating the position of meshes rendered by threejs, you are rather updating it based on a virtual physics world created by the engine.

```js
let scene, camera, renderer, cube, physicsCube;

function init() {
  (...)
}

function initPhysics() {
    const world = new CANNON.World(...);
    const cubeShape = new CANNON.Box(...)
    const cubeBody = new CANNON.Body({ mass: 5, shape: cubeShape })
    cubeBody.position.set(0, 5, 0)
    world.addBody(cubeBody)
}

function animate() {
  requestAnimationFrame(animate);
  cube.position.copy(physicsCube.position);
  cube.quaternion.copy(physicsCube.quaternion)
  renderer.render(scene, camera);
}
```

Now we have an infinitely falling cube, better add a floor, which is in fact nothing more then a rotated plane with mass of zero.

```js
const floorBody = new CANNON.Body({ mass: 0, shape: new CANNON.Plane() });
floorBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
world.addBody(floorBody);
```

Since the mass is zero, the plane is considered static and therefore there is no need to synchonize it with the rendered world. Not very elegant solution but we can simply add a `new THREE.PlaneGeometry()` in the init function. The example above works, but th

So how to make it better?
