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
        <canvas ref="canvas" id="canvas" width="640" height="480"></canvas>
      </v-flex>
    </v-layout>
    <v-container grid-list-xl fluid>
      <v-layout row wrap align-center justify-center>
        <v-flex v-for="(c,idx) in captures" :key="idx"  xs2 d-flex :class="{'choose':c.choose}" class="imgBox" >
          <v-card v-on:click="choose(idx)" flat class="d-flex">
            <v-img :src="c.img"  alt="."class="grey lighten-2" />
          </v-card>

        </v-flex>
      </v-layout>
    </v-container>


  </v-container>

</template>
<script>
export default {
  data() {
    return {
      video: {},
      canvas: {},
      captures: [],
      choose_idx:[]
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
      this.captures.push({
        img:this.canvas.toDataURL('image/png'),
        choose:false
      })
    },
    choose(idx) {
      console.log('cccc')
      this.captures[idx].choose = !this.captures[idx].choose
    },

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

</style>
