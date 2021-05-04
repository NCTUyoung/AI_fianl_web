<template>
  <v-container fluid grid-list-xl>
    <v-layout row wrap align-center justify-center>
      <v-flex shrink >
        <v-btn id="snap" v-on:click="capture()" class="green" :disabled="runcheck" >Snap Photo</v-btn>
        <v-btn id="rm" v-on:click="remove()" class="red">Remove</v-btn>
        <v-btn id="v" v-on:click="loadModel()" class="orange" :disabled="load">loadModel</v-btn>
<!--        <v-btn id="m" v-on:click="runEnhance()" class="orange" :disabled="runcheck">RunModel</v-btn>-->
        <canvas ref="canvas" id="canvas" width="256" height="256"></canvas>
        <v-img id="i"></v-img>
      </v-flex>
    </v-layout>
    <v-layout row wrap align-center justify-center >
<!--        <div class="float-crop">aaaaaaaaaaaaaaaaaaa</div>-->
        <h2>Camera Video Stream</h2>
        <v-flex  md12>
          <video ref="video" id="video" width="256" height="256" autoplay >
          </video>
        </v-flex>

        <v-flex md12>
          <canvas ref="output" id="output" width="256" height="256"></canvas>
        </v-flex>
      </v-layout>




    <v-container grid-list-xl fluid>
      <v-layout row wrap align-center justify-center>

        <v-flex v-for="(c,idx) in captures" :key="c.img"  xs12 sm12 md2  d-flex :class="{'choose':c.choose}" class="imgBox" >
          <v-card v-on:click="choose(idx)" flat class="d-flex">
            <v-img :src="c.img"  alt="."class="grey lighten-2" />
          </v-card>
        </v-flex>

      </v-layout>
    </v-container>

    <v-container grid-list-xl fluid>
      <v-layout row wrap align-center justify-center>

        <v-flex v-for="(c,idx) in transform" :key="c.img"  xs12 sm12 md2  d-flex :class="{'choose':c.choose}" class="imgBox" >
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
        transform:[],
        captures: [],
        choose_idx:[],
        imgData:[],
        outImage: {},
        session:{},
        load:false,
        finish:true,
        output_canvas:{}
      }
    },
    async  mounted() {
      this.video = this.$refs.video
      this.output_canvas = this.$refs.output
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
    computed: {
      runcheck: function() {
        return !this.load || !this.finish
      }
    },
    methods: {
      capture() {
        this.canvas = this.$refs.canvas

        this.canvas.getContext("2d").drawImage(this.video, 0, 0, 256, 256);
        // this.imgData =  this.canvas.getContext("2d").getImageData(10, 10, 50, 50);
        this.captures.push({
          img:this.canvas.toDataURL('image/png'),
          choose:false
        })
        this.runEnhance()
      },
      choose(idx) {
        this.captures[idx].choose = !this.captures[idx].choose
        this.transform[idx].choose = !this.transform[idx].choose
        this.choose_idx.push(idx)
      },
      remove() {
        this.captures = this.captures.filter((value,index,arr) =>{
          console.log(value)
          return value.choose === false
        })
        this.transform = this.transform.filter((value,index,arr) =>{
          console.log(value)
          return value.choose === false
        })
        this.choose_idx.length = 0
      },
      async loadModel() {
        // create a session
        this.session = new InferenceSession({ backendHint: "wasm" });
        // load the ONNX model file
        await this.session.loadModel('fast_depth.onnx')
        console.log('load finished !')
        this.load = true
      },
      async runEnhance() {
        this.finish = false
        let ctx = this.canvas.getContext("2d")
        let c = document.getElementById("output")
        let ct = c.getContext("2d")
        const imageData = ctx.getImageData(0, 0, 256, 256);
        ct.putImageData(imageData,0, 0)
        const { data, width, height } = imageData;
        // data processing
        const dataTensor = ndarray(new Float32Array(data), [width, height,4]);
        const dataProcessedTensor = ndarray(new Float32Array(width * height * 3), [1, 3, width, height]);
        ops.assign(dataProcessedTensor.pick(0, 0, null, null), dataTensor.pick(null, null, 0));
        ops.assign(dataProcessedTensor.pick(0, 1, null, null), dataTensor.pick(null, null, 1));
        ops.assign(dataProcessedTensor.pick(0, 2, null, null), dataTensor.pick(null, null, 2));
        ops.divseq(dataProcessedTensor, 255.0);
        const tensor = [new Tensor(new Float32Array(dataProcessedTensor.data), 'float32', [1, 3, width, height])];
        console.log(tensor)
        // tensor[0].data.set(dataProcessedTensor.data);
        // create a session

        const outputMap = await this.session.run(tensor);
        const outputTensor = outputMap.values().next().value;
        console.log(outputTensor)
        const dataOutput_3 = ndarray(new Float32Array(outputTensor.data), [3,width, height]);
        const dataOutput_4 = ndarray(new Float32Array(width*height*4), [width, height, 4]);
        ops.assign(dataOutput_4.pick(null, null, 0), dataOutput_3.pick(0,null, null));
        ops.assign(dataOutput_4.pick(null, null, 1), dataOutput_3.pick(1,null, null));
        ops.assign(dataOutput_4.pick(null, null, 2), dataOutput_3.pick(2,null, null));
        ops.mulseq(dataOutput_4,255.0)
        ops.assigns(dataOutput_4.pick(null, null, 3), 255.0);
        this.outImage = new ImageData(new Uint8ClampedArray(dataOutput_4.data), width, height)
        ct.putImageData(this.outImage,0, 0)

        this.finish = true
        this.transform.push({
          img:this.output_canvas.toDataURL('image/png'),
          choose:false
        })

      }
    }
  }
</script>

<style scoped>
  .container{
    padding: 12px;
  }
  canvas{
    background: darkolivegreen;
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
  #output {
    background-color: #000000;
    height:256px;
    width: 256px;
    display: none;
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
  .video-crop{
    background: mediumseagreen;


  }


  .float-crop {
    position: fixed;
    top: 50%;
    left: 50%;
    z-index: 1;
  }

</style>
