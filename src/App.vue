<template>
  <div id="app">
    <button class="btn1" @click="toLogin">Log In</button>
    <button class="btn2" @click="toLogout">Log Out</button><br>
    <doc-note
      class="code-outsize1"
      :width="width"
      height="300px"
      language="javascript"
      editable
      :foldable="true"
      :unfold="true"
      :codes="code1"
      @submit="submit1"
    />
    <doc-note
      class="code-outsize2"
      :width="width"
      language="javascript"
      editable
      :foldable="true"
      :unfold="false"
      :codes="code2"
      @submit="submit2"
    />
  </div>
</template>

<script>
import store from './store'

const dispatch = store.dispatch

export default {
  data() {
    return {
      width: '40%',
      code1: [],
      code2: [],
    }
  },
  methods: {
    toLogin() {
      dispatch('toLogin', true).then(res => { // 模拟接口-登录
        alert('You are now logged in!')
      })
    },
    toLogout() {
      dispatch('toLogin', false).then(res => {
        alert('You are now logged out!')
      })
    },
    submit1({data, close}) {
      console.log('submitData1', data) // data就是所有提交的数据
      dispatch('submit_code1', data).then(() => {
        close() // 初始化编辑状态
        dispatch('getCode1').then(res => {
          this.code1 = res
        })
      }).catch(err => {
        console.error(err)
      })
    },
    submit2({data, close}) {
      console.log('submitData2', data)
      dispatch('submit_code2', data).then(() => {
        close()
        dispatch('getCode2').then(res => {
          this.code2 = res
        })
      }).catch(err => {
        console.error(err)
      })
    },
  },
  created() {
    dispatch('getCode1').then(res => { // 模拟接口-获取code1数据
      console.log(res)
      this.code1 = res
    })

    dispatch('getCode2').then(res => { // 模拟接口-获取code2数据
      this.code2 = res
    })
  },
  mounted() {

  },
}
</script>

<style>
#app { width: 100%; overflow: hidden; position: relative; }
.btn1 { margin-left: 10%; margin-top: 20px; }
.btn2 { margin-left: 20px; margin-top: 20px; }
.code-outsize1 { margin-left: 10%; margin-top: 20px; margin-bottom: 50px; float: left; }
.code-outsize2 { margin-left: 20px; margin-top: 20px; margin-bottom: 50px; float: left; }
</style>
