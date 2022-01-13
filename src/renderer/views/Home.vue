<template>


  <div id="layerForm">
    
    <section class="hero is-dark is-bold">
      <div class="hero-body">
        <br />
        <div class="block has-text-centered">
          <figure class="image is-128x128">
            <!-- <img alt="Placeholder image" src="../../../_icons/512pxblue.png" /> -->
            <img src="https://hamonikr.org/layouts/hamonikr-join2/images/logo.png">
          </figure>
        </div>
        
        <div class="content has-text-centered">
          <h1 class="title is-4">Hamonikr - OS (Kiosk)</h1>
            
            <div id="licenseInfoLayer" class="text-center orange">
              Hamonikr OS 어쩌구 저쩌구..<br />
              기술지원을 받기 위해서는 라이선스 등록이 필요합니다.
            </div>

          <md-field> <label>라이선스 입력</label> <md-input v-model="licenseTxt" @click="licenseTextClear()" :disabled='isActive'></md-input> <md-button class="md-primary md-raised" @click="licenseSubmit()" :disabled='isActive'>Submit</md-button> </md-field>
          {{pushMessage}}
         <p v-if="errorshow">
          <ul>
            <li v-for="error in errors">{{error.text}}</li>
          </ul>
         </p>
        </div>

          <RouterLink class="button is-primary" to="about">
            <i class="material-icons">info</i>
            <span>About</span>
          </RouterLink>
          <RouterLink class="button is-primary" to="help">
            <i class="material-icons">help</i>
            <span>Help</span>
          </RouterLink>
          <br /><br /><br />
      </div>
    </section>
  </div>
</template>




<script lang="ts">

import Vue from 'vue'
import { ipcRenderer } from 'electron'
import axios from 'axios';

const vueLog = require('electron-log');
vueLog.transports.file.resolvePath = () => process.cwd() + '/logs/'+__dirname+'log.log';


enum TEST {}




export default Vue.extend({
  name: 'Home',
  
  data() { 
    return { 
      licenseTxt: '', 
      pushMessage: '',
      SearchResults:[],
      polling: null,
      isActive: false,
      errors: [
        { text: '' }
      ],
      errorshow:false,
    } 
  }, 
  created() {
      // 라이선스 파일 여부 체크 return Y/N
      ipcRenderer.on("readLiecenseFileResult", (event, args) => {
        vueLog.info(`Application Init] - License Check Result is :: ${args}`);
        if( args === 'Y' ){
          this.pushMessage = "하모니카 라이선스 인증이 완료되었습니다.";
          this.isActive = true;
        }
      });
      ipcRenderer.send("readLiecenseFile", "");

      // this.pollData();

  },
   mounted() {
      setInterval(this.pollData, 3000);
    },
  methods: { 
    pollData () {
      vueLog.info("aaaaaaaaaa");
      ipcRenderer.send("ChkLicenseProc", "");
      
      ipcRenderer.on("ChkLicenseProcResult", (event, args) => {
        vueLog.info(`ChkLicenseProcResult] - :: ${args}`);
        if( args === 'N' ){
          this.pushMessage = "Hamonikr-OS (KIOSK) License 인증을 진행해 주세요..";
          this.isActive = false;
        }
      });

    },
    isComplete(){
      return true;
    },
    licenseTextClear(){
      this.errors = [];
      this.errorshow = false;
    },
    clearAll(){
      this.licenseTxt = '';
    }, 
    validationCheck(){
      this.errors = [];
      vueLog.info(`Input License Number is :: ${this.licenseTxt} , Length() is :: ${this.licenseTxt.length}`);

      if( this.licenseTxt.length == 0 ){
        this.errors.push( { text: '라이선스 번호를 입력해주세요' } );
        this.errorshow = true;
        return ;
      }else{
        this.sendLicenseNumber();
      }
    },
    sendLicenseNumber(){
      var vm = this;
      // Step 1. Get SystemInfo ( Machineid + App UUID)
      ipcRenderer.send("machineIdAsync", "");

      // Step 2. Check License Number 
      ipcRenderer.once("isOsMachineIdData", (event, args) => {
        
        vueLog.info(`License Check Step 1 ] - System Info Data is :: ${args}`);

        let chkNecessaryData = args.split(":");
        vueLog.info(`License Check Step 1-1 ] - System Info Data is :: ${chkNecessaryData.length}`);

        if( chkNecessaryData.length < 2 ){
          vm.pushMessage = '라이선스 인증오류. 관리자에게 문의 바랍니다.';
          // vm.clearAll();
        } else {
          axios.post('http://192.168.0.118:8090/restapi/licenseAddProc', {
                    license_no: vm.licenseTxt.trim() , machineId_val: chkNecessaryData[0], appuuid: chkNecessaryData[1]
                  })
                  .then( function (response)
                  {
                    var jsonData = response.data;
                    vueLog.info(`License Check Step 1-2 ] - License Check Result is :: ${JSON.stringify(jsonData)}`);
                    if( jsonData.output == 'N' ){
                      vm.pushMessage = '잘못된 라이선스 번호입니다.';
                      vm.clearAll();
                    }else if( jsonData.output == 'DN' ){
                      vm.pushMessage = '라이선스 기간이 완료된 라이선스입니다.';
                      vm.clearAll();
                    }else if( jsonData.output == 'DUN' ){
                      vm.pushMessage = '사용중인 라이선스입니다..';
                      vm.clearAll();
                    }else if( jsonData.output == 'Y' ){
                      ipcRenderer.send('licenseSubmitProc', jsonData.lcnsno.trim());
                    }
                  })        
                  .catch(function (error) {
                    console.log(error);
                  })
        }
      });
      // Step 3. License Data Save 
      ipcRenderer.once("isOsMachineIdData", (event, args) => {
        this.pushMessage = "라이선스 등록이 정상처리 되었습니다.";
        setTimeout(function(){
          location.reload();
        },3000);
      });
      
    },
    licenseSubmit () { 
      this.validationCheck();
    },
    ttt(){
      ipcRenderer.once("abcd", (event, args) => {
        
      });
    }
  }
})
</script>


<style>
.hero-body {
  height: 100vh;
}

</style>
