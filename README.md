# WavyShaders
This shader code creates a dynamic, dreamy visual distortion effect on an image in a p5.js sketch. It uses a custom vertex and fragment shader.
The vertex shader passes texture coordinates to the fragment shader, which does the heavy lifting. 
Within the fragment shader, it applies a combination of flowing displacement and swirling distortion using fractal Brownian motion (FBM), a technique that layers noise patterns to create smooth, organic motion. 
The image texture is warped by this animated noise to simulate a kind of fluid, wave-like motion. It then overlays soft glowing "blobs" of color—shifting between sky blue and pink over time—based on additional FBM patterns. 
These blobs are subtly added to the image, giving it a shimmering, ethereal quality. 

let img;
let shaderProgram;
let vertShader = `
  attribute vec3 aPosition;
  attribute vec2 aTexCoord;
  varying vec2 vTexCoord;
  void main() {
    vTexCoord = aTexCoord;
    gl_Position = vec4(aPosition, 1.4);
  }
`;

let fragShader = `
  precision mediump float;
  uniform sampler2D uTexture;
  uniform float uTime;
  varying vec2 vTexCoord;
  float random(vec2 st) {
    return fract(sin(dot(st.xy, vec2(12.9898,78.233))) * 43758.5453123);
  }
  
  float noise(vec2 st) {
    vec2 i = floor(st);
    vec2 f = fract(st);
    vec2 u = f*f*(3.0-2.0*f);
    float a = random(i);
    float b = random(i + vec2(1.0, 0.0));
    float c = random(i + vec2(0.0, 1.0));
    float d = random(i + vec2(1.0, 1.0));
    return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
  }
  float fbm(vec2 st) {
    float value = 0.0;
    float amplitude = 0.5;
    for (int i = 0; i < 7; i++) {
      value += amplitude * noise(st);
      st *= 2.3;
      amplitude *= 0.5;
    }
    return value;
  }
  vec2 swirl(vec2 uv, float strength) {
    float angle = fbm(uv * 2.0 + uTime * 0.1) * 6.2831; // 2*PI swirl
    float s = sin(angle);
    float c = cos(angle);
    uv -= 0.5;
    uv = mat2(c, -s, s, c) * uv;
    uv += 0.5;
    return mix(vTexCoord, uv, strength);
  }
  void main() {
    vec2 uv = vTexCoord;
    // big slow flow
    vec2 flow = vec2(
      fbm(uv * 2.0 + vec2(uTime * 0.03, uTime * 0.04)),
      fbm(uv * 2.0 - vec2(uTime * 0.05, uTime * 0.02))
    );
   uv += (flow - 0.5) * 0.4;
  // apply swirling
    uv = swirl(uv, 0.25);
// sample the texture
    vec4 tex = texture2D(uTexture, uv);
 // glowing blobs
    float blobs = fbm(uv * 5.0 + vec2(uTime * 0.1, uTime * 0.12));
    blobs = smoothstep(0.4, 0.8, blobs);
// dynamic soft coloring
    vec3 blobColor = mix(
      vec3(0.5, 0.7, 1.0),   // sky blue
      vec3(1.0, 0.6, 0.9),   // soft pink
      sin(uTime * 0.5) * 0.8 + 0.8
    );
 // apply the blobs softly
    tex.rgb += blobColor * blobs * 0.45;
 gl_FragColor = vec4(tex.rgb, 2.0);
  }`;
function preload() {
  img = loadImage('backgroundformushroom.png');
}
function setup() {
  createCanvas(1800, 1500, WEBGL);
  shaderProgram = createShader(vertShader, fragShader);
  noStroke();
}
function draw() {
  shader(shaderProgram);
  shaderProgram.setUniform('uTime', millis() / 850.0);
  shaderProgram.setUniform('uTexture', img);
  rect(-width/2, -height/2, width, height);
}
