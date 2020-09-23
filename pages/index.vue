<template>
  <div id="dwt-container">
    <Viewer 
      :id="this.viewer.id" 
      :ref="this.viewer.id"
      :dwtRef="this.dwt.obj"
      :width="this.viewer.width"
      :height="this.viewer.height"
    >
      <!-- DWT viewer -->
    </Viewer>
    <div>
      <v-tabs id="dwt-control">
        <v-tab :key="0" @click="handleTabChange(0)">Scan</v-tab>
        <v-tab :key="1" @click="handleTabChange(1)">Webcam Capture</v-tab>
        <v-tab :key="2" @click="handleTabChange(2)">OCR</v-tab>
        <v-tab-item :key="0">
          <Scan :ref="'scan-panel'"></Scan>
        </v-tab-item>
        <v-tab-item :key="1">
          <Webcam :ref="'webcam-panel'"></Webcam>
        </v-tab-item>
        <v-tab-item>
          <OCR :ref="'ocr-panel'"/>
        </v-tab-item>
      </v-tabs>
      <div>
        <h4>File Upload</h4>
        <v-text-field 
          v-model="upload.fileName"
          placeholder="File Name"
        ></v-text-field>
        <v-select
          v-model="upload.selectFormat"
          :items="upload.formats"
          item-text="text"
          item-value="id"
          label="Format select"
          outlined
        >
        </v-select>
        <v-btn @click="uploadFile">Upload</v-btn>
      </div>
    </div>
  </div>
</template>

<script>
import dwt from 'dwt'
import Viewer from '~/components/Viewer'
import Scan from '~/components/Scan'
import Webcam from '~/components/Webcam.vue'
import OCR from '~/components/OCR.vue'

export default {
  name: 'dwt',
  components: {
    Viewer,
    Scan,
    Webcam,
    OCR
  },
  data () {
    return {
      dwt: {
        obj: null,
        id: 'dwtObject',
        licenseKey: 't01016QAAADyBe7yfb9oPaRKoDodUi2D6w3Dj/XeSforvLiBX6PXItwyqx3NL/4Uso1U/t4Gol58RCjB9B1q+RjxJ2qOVHa1eGzRmGbzga3PGGn1/tDAWpk/DKsyhQmO9F1PDDdxIL+c='
      },
      viewer: {
        id: 'dwtViewer',
        width: '100%',
        height: '600px'
      },
      upload: {
        fileName: '',
        formats: [
          {id: 'pdf', text: 'pdf'},
          {id: 'jpg', text: 'jpg'},
          {id: 'tif', text: 'tif'}
        ],
        selectFormat: ''
      }
    }
  },
  mounted () {
    this.mountDWT()
      .then(() => {
        // this.handleTabChange(0)
        this.initPanels()
        this.bindViewer()
      })
  },
  methods: {
    bindViewer () {
      this.$refs[this.viewer.id].mountViewer(this.dwt.obj)
    },
    mountDWT () {
      return new Promise((res, rej) => {
        this.unmountDWT()
          .then(() => {
            dwt.WebTwainEnv.UseLocalService = true;
            dwt.WebTwainEnv.ResourcesPath = "/dwt";
            dwt.WebTwainEnv.ProductKey = this.dwt.licenseKey
            dwt.WebTwainEnv.AutoLoad = false;
            dwt.WebTwainEnv.Containers = [];
            dwt.WebTwainEnv.IfAddMD5InUploadHeader = false;
            dwt.WebTwainEnv.IfConfineMaskWithinTheViewer = false;
            let dwtConfig = { WebTwainId: this.dwt.id }
            // By default, use local service is true
            dwt.WebTwainEnv.CreateDWTObjectEx(
                  dwtConfig, 
                  (dwtObject) => { this.dwt.obj = dwtObject; res(true);},
                  (errStr) => { console.log(`failed to initialize dwt, message: ${errStr}`); rej(false);}
            )
          })
      })
    },
    /**
     * Delete dwt instance
     */
    unmountDWT () {
      return new Promise((res, rej) => {
        if (dwt.WebTwainEnv.DeleteDWTObject(this.dwt.id)) {
          res(true)
        } else {
          rej(false)
        }
      })
    },
    initPanels () {
      window.refs = this.$refs
      this.$refs['scan-panel'].setupScan(this.dwt.obj)
    },
    handleTabChange (key) {
      let loadOnSwitch = () => {
        switch(key) {
          case 0: { this.$refs['scan-panel'].setupScan(this.dwt.obj); break; }
          case 1: { this.$refs['webcam-panel'].setupWebcam(dwt.Lib.detect.ssl, dwt.EnumDWT_VideoRotateMode, this.dwt.obj); break; }
          case 2: { this.$refs['ocr-panel'].setupOcr(dwt.WebTwainEnv.ResourcesPath, true, this.dwt.obj, dwt.Lib); break; }
        }
      }
      setTimeout(loadOnSwitch, 100)
    },
    uploadFile () {
    const path = 'localhost:3000/api/File'
    let uploadFileName = this.upload.fileName + '.' + this.upload.selectFormat
    let fileType = ((select) => {
          switch (select) {
              case 'jpg': { return dwt.EnumDWT_ImageType.IT_JPG }
              case 'pdf': { return dwt.EnumDWT_ImageType.IT_PDF }
              case 'tif': { return dwt.EnumDWT_ImageType.IT_TIF }
          }
    })(this.upload.selectFormat)

    let uploadFormat = dwt.EnumDWT_UploadDataFormat.Binary

    const DWObj = this.dwt.obj
    if (DWObj) {
      // DWObj.HTTPPort = port
      // DWObj.IfSSL = true
      let indices = DWObj.SelectedImagesIndices
      DWObj.HTTPUpload(
          path,
          indices,
          fileType,
          uploadFormat,
          uploadFileName,
          () => { alert('success') },
          (errCode, errStr, res) => {
              console.error(`${errCode}: ${errStr}. Server return: ${ res }`)
          }
      )
    }
  }
  },
  
}
</script>

<style scoped>
#dwt-container {
  display: flex;
  height: inherit;
  width: inherit;
}
#dwt-control {
  max-width: 400px;
}
</style>