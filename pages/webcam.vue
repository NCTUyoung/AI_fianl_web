<template>
  <v-container>
    <v-layout row wrap align-center justify-center>
      <v-flex xs12>
        <video ref="video" id="video" width="640" height="480" autoplay ></video>
      </v-flex>
    </v-layout>
    <v-layout row wrap align-center justify-center>
      <v-flex shrink >
        <v-btn id="snap" v-on:click="capture()" class="green">Snap Photo</v-btn>
        <v-btn id="rm" v-on:click="remove()" class="red">Remove</v-btn>
        <v-btn id="m" v-on:click="runEnhance()" class="orange">RunModel</v-btn>
        <canvas ref="canvas" id="canvas" width="640" height="480"></canvas>
      </v-flex>
    </v-layout>
    <v-container grid-list-xl fluid>
      <v-layout row wrap align-center justify-center>

          <v-flex v-for="(c,idx) in captures" :key="c.img"  xs2 d-flex :class="{'choose':c.choose}" class="imgBox" >
            <v-card v-on:click="choose(idx)" flat class="d-flex">
              <v-img :src="c.img"  alt="."class="grey lighten-2" />
            </v-card>
          </v-flex>

      </v-layout>
    </v-container>


  </v-container>

</template>
<script>
import ndarray from 'ndarray';
import ops from 'ndarray-ops';
import { Tensor, InferenceSession } from "onnxjs";
export default {

  data() {
    return {
      video: {},
      canvas: {},
      captures: [],
      choose_idx:[],
      imgData:[]
    }
  },
  mounted() {
    this.video = this.$refs.video
    console.log(navigator.mediaDevices)
    console.log(navigator.mediaDevices.getUserMedia)
    if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
      console.log("Something went wrong!");
      navigator.mediaDevices.getUserMedia({ video: true }).then(stream => {
        this.video.srcObject = stream
        console.log(stream)
        this.video.play()
      }).catch(function (err0r) {
        console.log("Something went wrong!");
      });
    }
  },
  methods: {
    capture() {
      this.canvas = this.$refs.canvas
      this.canvas.getContext("2d").drawImage(this.video, 0, 0, 640, 480);
      // this.imgData =  this.canvas.getContext("2d").getImageData(10, 10, 50, 50);

      this.captures.push({
        img:this.canvas.toDataURL('image/png'),
        choose:false
      })
    },
    choose(idx) {
      this.captures[idx].choose = !this.captures[idx].choose
      this.choose_idx.push(idx)
    },
    remove() {
      this.captures = this.captures.filter((value,index,arr) =>{
        console.log(value)
        return value.choose === false
      })
      this.choose_idx.length = 0

    },
    async runEnhance() {
      let ctx = this.canvas.getContext("2d")
      const imageData = ctx.getImageData(0, 0, 320, 240);
      const { data, width, height } = imageData;

      // data processing
      const dataTensor = ndarray(new Float32Array(data), [width, height, 3]);
      const dataProcessedTensor = ndarray(new Float32Array(width * height * 3), [1, 3, width, height]);
      ops.assign(dataProcessedTensor.pick(0, 0, null, null), dataTensor.pick(null, null, 0));
      ops.assign(dataProcessedTensor.pick(0, 1, null, null), dataTensor.pick(null, null, 1));
      ops.assign(dataProcessedTensor.pick(0, 2, null, null), dataTensor.pick(null, null, 2));
      ops.divseq(dataProcessedTensor, 255);

      const tensor = [new Tensor(new Float32Array(width * height * 3), 'float32', [1, 3, width, height])];
      console.log(tensor)
      tensor[0].data.set(dataProcessedTensor.data);

      // create a session
      const session = new InferenceSession({ backendHint: "wasm" });
      // load the ONNX model file
      const model = await session.loadModel('/unet_fast.onnx')
      const outputMap = await session.run(tensor);
      console.log(model)
      const outputTensor = outputMap.values().next().value;
      console.log(outputTensor)


    }

  }
}
</script>

<style scoped>
.container{
padding: 12px;
}
body {
  background-color: #f0f0f0;
}
#app {
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
#video {
  background-color: #000000;
  width: 100%;
}
#canvas {
  display: none;
}
ul{
  display: flex;
  flex-direction: row;
  flex-wrap: nowrap
}
li {

  display: flex;

  padding: 5px;
}
.imgBox.choose{
  border-color: mediumseagreen;
}
.imgBox {
  background: transparent;
  margin: 2px;
  border-color: transparent;
  border-width:4px;
  border-style:solid;
  border-radius: 10px;
}
.list-enter-active, .list-leave-active {
  transition: all 1s;
}
.list-enter, .list-leave-to /* .list-leave-active below version 2.1.8 */ {
  opacity: 0;
  transform: translateY(30px);
}
</style>
