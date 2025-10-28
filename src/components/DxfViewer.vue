<template>
<div class="canvasContainer" ref="canvasContainer">
    <q-inner-loading :showing="isLoading" color="primary" style="z-index: 10"/>
    <div v-if="progress !== null" class="progress">
        <q-linear-progress color="primary" :indeterminate="progress < 0" :value="progress" />
        <div v-if="progressText !== null" class="progressText">{{progressText}}</div>
    </div>
    <div v-if="error !== null" class="error" :title="error">
        <q-icon name="warning" class="text-red" style="font-size: 4rem;" /> Error occurred: {{error}}
    </div>
</div>
</template>

<script>
import {DxfViewer} from "dxf-viewer"
import * as three from "three"
import {GLTFExporter} from "three/examples/jsm/exporters/GLTFExporter"
import DxfViewerWorker from "worker-loader!./DxfViewerWorker"

const gltfExporter = new GLTFExporter()
const link = document.createElement('a')
/** Events: all DxfViewer supported events (see DxfViewer.Subscribe()), prefixed with "dxf-". */
export default {
    name: "DxfViewer",
    inject:['triggerExportGLTF'],
    props: {
        dxfUrl: {
            default: null
        },
        /** List of font URLs. Files should have TTF format. Fonts are used in the specified order,
         * each one is checked until necessary glyph is found. Text is not rendered if fonts are not
         * specified.
         */
        fonts: {
            default: null
        },
        options: {
            default() {
                return {
                    clearColor: new three.Color("#000"),
                    autoResize: true,
                    colorCorrection: true,
                    sceneOptions: {
                        wireframeMesh: true
                    }
                }
            }
        }
    },

    data() {
        return {
            isLoading: false,
            progress: null,
            progressText: null,
            curProgressPhase: null,
            error: null
        }
    },

    watch: {
        async dxfUrl(dxfUrl) {
            if (dxfUrl !== null) {
                await this.Load(dxfUrl)
            } else {
                this.dxfViewer.Clear()
                this.error = null
                this.isLoading = false
                this.progress = null
            }
        }
    },

    methods: {
        async Load(url) {
            this.isLoading = true
            this.error = null
            try {
                await this.dxfViewer.Load({
                    url,
                    fonts: this.fonts,
                    progressCbk: this._OnProgress.bind(this),
                    workerFactory: DxfViewerWorker
                })
            } catch (error) {
                console.warn(error)
                this.error = error.toString()
            } finally {
                this.isLoading = false
                this.progressText = null
                this.progress = null
                this.curProgressPhase = null
                

                // console.log(this.GetViewer().GetOrigin())
            }
        },
        exportGLTF(name='dxfviewer') {
            const scene = this.GetViewer().GetScene()
            const group = new three.Group();
            const layerMatMap = {};
                scene.children.forEach(mesh => {
                    const userData= {
                        dxfLayer : mesh._dxfViewerLayer.name ,
                        displayName :  mesh._dxfViewerLayer.displayName 
                    }
                    const key = userData.dxfLayer
                    let newObject = null;
                    if(!layerMatMap[key]){
                        layerMatMap[key] = new three.MeshBasicMaterial({color: mesh._dxfViewerLayer.color})
                    }
                    mesh.material = layerMatMap[key]
                    
                    
                    if(mesh.isMesh){
                        newObject= new three.Mesh(mesh.geometry,layerMatMap[key])
                    }
                    else if(mesh.isLineSegments){
                        newObject= new three.LineSegments(mesh.geometry,layerMatMap[key])
                    }
                    else if(mesh.isPoints){
                        newObject= new three.Points(mesh.geometry,layerMatMap[key])
                    }
                    if(newObject){
                        newObject.userData = userData
                        group.add(newObject)
                    }
                })
                console.log(group);
                
            gltfExporter.parse(group,(result)=>{
                URL.revokeObjectURL(link.href)
                link.href = URL.createObjectURL(new Blob([result], {type: "application/octet-stream"}))
                link.download = `${name}_${Date.now()}`+ ".glb"
                link.click()
            },()=>{
                console.error("Error during glTF export")
            },{binary:true})
        },

        /** @return {DxfViewer} */
        GetViewer() {
            return this.dxfViewer
        },

        _OnProgress(phase, size, totalSize) {
            if (phase !== this.curProgressPhase) {
                switch(phase) {
                case "font":
                    this.progressText = "Fetching fonts..."
                    break
                case "fetch":
                    this.progressText = "Fetching file..."
                    break
                case "parse":
                    this.progressText = "Parsing file..."
                    break
                case "prepare":
                    this.progressText = "Preparing rendering data..."
                    break
                }
                this.curProgressPhase = phase
            }
            if (totalSize === null) {
                this.progress = -1
            } else {
                this.progress = size / totalSize
            }
        }
    },

    mounted() {
        this.dxfViewer = new DxfViewer(this.$refs.canvasContainer, this.options)
        const Subscribe = eventName => {
            this.dxfViewer.Subscribe(eventName, e => this.$emit("dxf-" + eventName, e))
        }
        for (const eventName of ["loaded", "cleared", "destroyed", "resized", "pointerdown",
                                 "pointerup", "viewChanged", "message"]) {
            Subscribe(eventName)
        }
    },

    destroyed() {
        this.dxfViewer.Destroy()
        this.dxfViewer = null
    }
}
</script>

<style scoped lang="less">

.canvasContainer {
    position: relative;
    width: 100%;
    height: 100%;
    min-width: 100px;
    min-height: 100px;

    .progress {
        position: absolute;
        z-index: 20;
        width: 90%;
        margin: 20px 5%;

        .progressText {
            margin: 10px 20px;
            font-size: 14px;
            color: #262d33;
            text-align: center;
        }
    }

    .error {
        width: 100%;
        height: 100%;
        position: absolute;
        z-index: 20;
        padding: 30px;

        img {
            width: 24px;
            height: 24px;
            vertical-align: middle;
            margin: 4px;
        }
    }
}

</style>
