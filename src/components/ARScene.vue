<template>
  <div id="ARScene"></div>
</template>

<script>
import * as THREE from 'three';
import { ArToolkitProfile, ArToolkitSource, ArToolkitContext, ArMarkerControls} from 'ar-js-org/three.js/build/ar-threex.js';

export default {
  name: "ARScene",
  mounted() {
    ArToolkitContext.baseURL = './'
    console.log("ARScene mounted");

    // init renderer
    var renderer	= new THREE.WebGLRenderer({
      antialias	: true,
      alpha: true
    });

    renderer.setClearColor(new THREE.Color('lightgrey'), 0)
    // renderer.setPixelRatio( 2 );
    renderer.setSize( window.innerWidth, window.innerHeight );
    renderer.domElement.style.position = 'absolute'
    renderer.domElement.style.top = '0px'
    renderer.domElement.style.left = '0px'
    document.body.appendChild( renderer.domElement ); // We should be able to specify an html element to append AR.js related elements to.

    // array of functions for the rendering loop
    var onRenderFcts= [];

    // init scene and camera
    var scene	= new THREE.Scene();

    //////////////////////////////////////////////////////////////////////////////////
    //		Initialize a basic camera
    //////////////////////////////////////////////////////////////////////////////////

    // Create a camera
    var camera = new THREE.Camera();
    scene.add(camera);
    const artoolkitProfile = new ArToolkitProfile();
    artoolkitProfile.sourceWebcam(); // Is there good reason for having a function to set the sourceWebcam but not the displayWidth/Height etc?
    
    // add existing parameters, this is not well documented
    let additionalParameters = {
        // Device id of the camera to use (optional)
        deviceId: null,
        // resolution of at which we initialize in the source image
        sourceWidth: 640,
        sourceHeight: 480,
        // resolution displayed for the source
        displayWidth: 640,
        displayHeight: 480,
    }
    
    Object.assign(artoolkitProfile.sourceParameters, additionalParameters);
    console.log(artoolkitProfile.sourceParameters); // now includes the additionalParameters

    const arToolkitSource = new ArToolkitSource(artoolkitProfile.sourceParameters);

    arToolkitSource.init(function onReady(){
      onResize();
    })

    // handle resize
    window.addEventListener('resize', function(){
      onResize(); 
    })

    // resize is not called for the canvas on init. The canvas with the cube seems to be resized correctly at start.
    // Is that maybe a vue-specific problem?
    function onResize(){
      arToolkitSource.onResizeElement()
      arToolkitSource.copyElementSizeTo(renderer.domElement)
      if(arToolkitContext.arController !== null){
        arToolkitSource.copyElementSizeTo(arToolkitContext.arController.canvas)
      }
    }
      

    ////////////////////////////////////////////////////////////////////////////////
    //          initialize arToolkitContext
    ////////////////////////////////////////////////////////////////////////////////

    // create atToolkitContext
    var arToolkitContext = new ArToolkitContext({
      debug: false,
      cameraParametersUrl: ArToolkitContext.baseURL + 'data/camera_para.dat',
      detectionMode: 'mono',
      canvasWidth: 640,
      canvasHeight: 490,
      imageSmoothingEnabled : true, // There is still a warning about mozImageSmoothingEnabled when using Firefox
    })
    
    // initialize it
    arToolkitContext.init(function onCompleted(){
      // copy projection matrix to camera
      camera.projectionMatrix.copy( arToolkitContext.getProjectionMatrix() );
    })

    // update artoolkit on every frame
    onRenderFcts.push(function(){
      if( arToolkitSource.ready === false )	return

      arToolkitContext.update( arToolkitSource.domElement )
    })

    ////////////////////////////////////////////////////////////////////////////////
    //          Create a ArMarkerControls
    ////////////////////////////////////////////////////////////////////////////////

    var markerGroup = new THREE.Group()
    scene.add(markerGroup)

    var markerControls = new ArMarkerControls(arToolkitContext, markerGroup, {
      type: 'pattern',
      patternUrl: ArToolkitContext.baseURL + 'data/patt.hiro',
      smooth: true,
      smoothCount: 5,
      smoothTolerance: 0.01,
      smoothThreshold: 2
    })

    //////////////////////////////////////////////////////////////////////////////////
    //		add an object in the scene
    //////////////////////////////////////////////////////////////////////////////////

    var markerScene = new THREE.Scene()
    markerGroup.add(markerScene)

    var mesh = new THREE.AxesHelper()
    markerScene.add(mesh)

    // add a torus knot
    var geometry	= new THREE.BoxGeometry(1,1,1);
    var material	= new THREE.MeshNormalMaterial({
      transparent : true,
      opacity: 0.5,
      side: THREE.DoubleSide
    });
    var mesh	= new THREE.Mesh( geometry, material );
    mesh.position.y	= geometry.parameters.height/2
    markerScene.add(mesh)

    var geometry	= new THREE.TorusKnotGeometry(0.3,0.1,64,16);
    var material	= new THREE.MeshNormalMaterial();
    var mesh	= new THREE.Mesh( geometry, material );
    mesh.position.y	= 0.5
    markerScene.add( mesh );

    onRenderFcts.push(function(delta){
      mesh.rotation.x += delta * Math.PI
    })

    //////////////////////////////////////////////////////////////////////////////////
    //		render the whole thing on the page
    //////////////////////////////////////////////////////////////////////////////////
    onRenderFcts.push(function(){
      renderer.render(scene, camera);
    })

      // run the rendering loop
    var lastTimeMsec = null;
    requestAnimationFrame(function animate(nowMsec){
      // keep looping
      requestAnimationFrame(animate);
      // measure time
      lastTimeMsec	= lastTimeMsec || nowMsec-1000/60
      var deltaMsec	= Math.min(200, nowMsec - lastTimeMsec)
      lastTimeMsec	= nowMsec
      // call each update function
      onRenderFcts.forEach(function(onRenderFct){
        onRenderFct(deltaMsec/1000, nowMsec/1000)
      })
    })
    }
}
</script>

<style>

</style>