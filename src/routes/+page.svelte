<script lang='ts'>
  import { onMount } from "svelte";
  import * as THREE from 'three';
  import * as CANNON from 'cannon-es';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

  let scene:THREE.Scene, camera:THREE.PerspectiveCamera, renderer:THREE.Renderer, track:THREE, car:THREE, world:CANNON.World, carBody:CANNON.Body;
  onMount(()=>{
    init();
    // Handle window resize
    window.addEventListener('resize', () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });
  });


  function init(){

    scene = new THREE.Scene();
    scene.background = new THREE.Color(0xffbb88);
    
    camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
    camera.position.set(0, 5,-10);
    
    // document.body.appendChild(renderer.domElement);
    const canvas = document.querySelector(".draw");
    renderer = new THREE.WebGLRenderer({canvas:canvas, antialias:true});
    renderer.setSize(window.innerWidth, window.innerHeight);

    //ambient light
    const ambientLight= new THREE.AmbientLight(0xffddbb);
    scene.add(ambientLight);

    //directional light
    const directionalLight = new THREE.DirectionalLight(0xffcc88, 1.8);
    directionalLight.position.set(5,10,5).normalize();
    scene.add(directionalLight);

    //Initialize Cannon.js physics world
    world = new CANNON.World();
    world.gravity.set(0, -9.82, 0);

    //set up ground plane 
    const groundBody = new CANNON.Body({
      type: CANNON.Body.STATIC,
      shape: new CANNON.Plane()
    });
    groundBody.quaternion.setFromEuler(-Math.PI/2, 0, 0); //Rotate to be horizontal
    world.addBody(groundBody);

    
    const loader = new GLTFLoader(); 
    //load track model
    loader.load('./AC-CIRCUIT.glb', (gltf)=>{
      track = gltf.scene;
      track.position.y = -0.1;
      scene.add(track);
    });
    //load car model
    loader.load('./porsche_911_GT.glb', (gltf)=>{
      car = gltf.scene;
      scene.add(car);
      car.position.set(0, 0.5,0);

      // Create Physics body for car
      const carShape = new CANNON.Box(new CANNON.Vec3(1,0.5,2));
      carBody = new CANNON.Body({
        mass:10,
        shape: carShape
      });
      carBody.position.set(0,0.5,0);

      carBody.angularDamping = 0.6; // Add some damping to reduce flipping
      carBody.linearDamping = 0.4; // Add some damping to smooth out movement
      carBody.updateMassProperties(); // Recalculate mass properties

      world.addBody(carBody);
    });

    // Handle keyboard input
    let keysPressed:any = {};
    window.addEventListener('keydown', (event) => {
      keysPressed[event.key] = true;
    });

    window.addEventListener('keyup', (event) => {
      keysPressed[event.key] = false;
    });

    let acceleration = 0;
    const maxAcceleration = 0.6;
    const accelerationStep = 0.01;
    const decelerationStep = 0.007;

    // Start the animation loop
    function animate() {
      requestAnimationFrame(animate);

      // Step the physics world
      world.step(1/60);

      // Apply forces based on user input
      if (carBody) {
        const forwardVector = new CANNON.Vec3(0,0,-1);
        carBody.quaternion.vmult(forwardVector, forwardVector); // Get forward direction

        if (keysPressed[" "]){
          acceleration = acceleration/2;
        }

        if (keysPressed['ArrowUp'] || keysPressed['w'] || keysPressed['W']) {
          acceleration = Math.min(acceleration+accelerationStep, maxAcceleration);
          // velocity = forwardVector.scale(0.32); // Move forward
        }
        else if (keysPressed['ArrowDown'] || keysPressed['s'] || keysPressed['S']) {
          acceleration = Math.max(acceleration-accelerationStep, -maxAcceleration);
          // velocity = forwardVector.scale(-0.32); // Move backward
        } else {
          // Gradual deceleration
          if (acceleration > 0) {
            acceleration = Math.max(acceleration - decelerationStep, 0);
          }else if (acceleration < 0) {
            acceleration = Math.min(acceleration + decelerationStep, 0);
          }
        }
        const velocity = forwardVector.scale(acceleration*0.8);

        // Apply the velocity to the car
        // carBody.velocity.x = velocity.x;
        // carBody.velocity.y = carBody.velocity.y; // Preserve existing vertical velocity (e.g., gravity)
        // carBody.velocity.z = velocity.z;
        
        // Apply the velocity to the car
        carBody.position.x = carBody.position.x + velocity.x;
        carBody.position.y = carBody.position.y + velocity.y;
        carBody.position.z = carBody.position.z + velocity.z;

        // Handle rotation
        if (keysPressed['ArrowLeft'] || keysPressed['a'] || keysPressed['A']) {
          carBody.angularVelocity.y = 16; //Turn left
          // carBody.applyTorque(new CANNON.Vec3(0, 20, 0)); // Turn left
        } else if (keysPressed['ArrowRight'] || keysPressed['d'] || keysPressed['D']) {
          carBody.angularVelocity.y = -16; //Turn right
          // carBody.applyTorque(new CANNON.Vec3(0, -20, 0)); // Turn right
        } else {
          carBody.torque.set(0,0,0); // Stop rotating if no turning keys are pressed
        }

        // Apply anti-roll force
        // const antiRollForce = 0.1;
        // if (carBody.position.y > 0.5) {
        //   carBody.applyForce(new CANNON.Vec3(0, -antiRollForce, 0), carBody.position);
        // }

        // // Gyroscopic Stabilizer
        // const upVector = new CANNON.Vec3(0, 1, 0);
        // const currentUp = new CANNON.Vec3();
        // carBody.quaternion.vmult(upVector, currentUp);

        // // Calculate angle between upVector and currentUp
        // const angle = Math.acos(upVector.dot(currentUp) / (upVector.length() * currentUp.length()));

        // const axis = new CANNON.Vec3();
        // currentUp.cross(upVector, axis);
        // axis.normalize();
        
        // const torque = axis.scale(angle * 0.5);
        // carBody.applyTorque(torque);

        // Prevent flipping by limiting rotation
        carBody.quaternion.x = 0;
        carBody.quaternion.z = 0;

        
        // Sync Three.js car model with Cannon.js physics body
        car.position.copy(carBody.position);
        car.quaternion.copy(carBody.quaternion);
      }
      renderer.render(scene, camera);

      // Follow the car with the camera
      if (car) {
        const relativeCameraOffset = new THREE.Vector3(0, 2, 4);
        const cameraOffset = relativeCameraOffset.applyMatrix4(car.matrixWorld);
        camera.position.lerp(cameraOffset, 0.1);  // Smooth following
        camera.lookAt(car.position);
      }
    }
    animate();
  };

</script>
