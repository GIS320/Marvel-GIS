<template>
    <div>
        <div class="query">
            <el-autocomplete
                    class="inline-input"
                    v-model="marvelid"
                    :fetch-suggestions="querySearch"
                    placeholder="输入名称查询"
                    @select="handleSelect"
            ></el-autocomplete>
        </div>
        <div id="3d-graph">
        </div>
        <el-checkbox v-model="selectAll" style="position: absolute;z-index: 5;top: 50px;left: 60px" @change="initGraph1Again" class = "selectAll">ALL</el-checkbox>
        <el-checkbox-group v-model="checkedList" style="position: absolute;z-index: 5;top: 70px;left: 60px;display:flex;flex-direction: column" @change="filterGraphData">
            <el-checkbox v-for="opt in checkList" :label="opt" :key="opt" class = "check"></el-checkbox>
        </el-checkbox-group>
    </div>
</template>

<script>
    import rawData from '../../assets/json/marvel-data.json';
    import * as THREE from 'three';
    import ForceGraph3D from '3d-force-graph';
    import {request} from "network/request";

    export default {
        name: "graph",
        data(){
          return{
              marvelid: "",
              idList: [],
              Graph: null,
              gData:null,
              checkList:['HERO','AVENGERS','SHIELD','ARMY','VILLAIN','HYDRA','TEN_RINGS','AIM','RAVAGERS','NOVA_CORP','BLACK_ORDER','STAN_LEE'],
              checkedList:[],
              selectAll:true
          }
        },
        mounted() {
            this.initGraph()
            this.loadData()
        },
        methods: {
            initGraph() {
                let characters = rawData["characters"];
                let relationship = rawData["relationship"];
                let characters_arr = [];
                let links = [];
                for (let key in characters) {
                    characters_arr.push(characters[key])
                    // console.log(characters[key])
                }
                for (let key in relationship) {
                    links.push({
                        "source": relationship[key]["id"],
                        "target": relationship[key]["target_id"],
                        "type": parseInt(relationship[key]["relationship"])
                    })
                }
                let gData = {
                    nodes: characters_arr,
                    links: links
                };
                //console.log(gData);
                const Graph = ForceGraph3D();
                Graph(document.getElementById('3d-graph'))
                    .linkAutoColorBy(d => d.type)
                    .nodeThreeObject(({id}) => {
                        // use a sphere as a drag handle
                        const obj = new THREE.Mesh(
                            new THREE.SphereGeometry(7),
                            new THREE.MeshBasicMaterial({depthWrite: false, transparent: true, opacity: 0})
                        );

                        // add img sprite as child
                        let imgurl = require(`../../assets/img/marvel_svg/${id}.svg`);
                        const imgTexture = new THREE.TextureLoader().load(imgurl);
                        const material = new THREE.SpriteMaterial({map: imgTexture});
                        const sprite = new THREE.Sprite(material);
                        sprite.scale.set(24, 24);
                        obj.add(sprite);

                        return obj;
                    })
                    .backgroundColor('#172734')
                    .onNodeClick(node=>{
                        // console.log(node);
                        this.$bus.$emit('clickItem',node.id)
                        const distance = 40;//自己设的，可以改
                        const distRatio = 1 + distance/Math.hypot(node.x, node.y, node.z);
                        Graph.cameraPosition(
                            { x: node.x * distRatio, y: node.y * distRatio, z: node.z * distRatio }, // new position
                            node, // lookAt ({ x, y, z })
                            3000  // 动画过渡时间（毫秒）
                        );
                    })
                    .graphData(gData);
                let scene=Graph.scene();
                // let axesHelper = new THREE.AxesHelper(100);
                // scene.add( axesHelper );
                let camera=Graph.camera();
                let renderer=Graph.renderer()
                let controls=Graph.controls()
                controls.maxDistance = 1000;
                let materialArray = [];
                let texture_ft = new THREE.TextureLoader().load( require('../../assets/img/skybox/space1/ft.png'));
                let texture_bk = new THREE.TextureLoader().load( require('../../assets/img/skybox/space1/bk.png'));
                let texture_up = new THREE.TextureLoader().load( require('../../assets/img/skybox/space1/up.png'));
                let texture_dn = new THREE.TextureLoader().load( require('../../assets/img/skybox/space1/dn.png'));
                let texture_rt = new THREE.TextureLoader().load( require('../../assets/img/skybox/space1/rt.png'));
                let texture_lf = new THREE.TextureLoader().load( require('../../assets/img/skybox/space1/lf.png'));

                materialArray.push(new THREE.MeshBasicMaterial( { map: texture_ft }));
                materialArray.push(new THREE.MeshBasicMaterial( { map: texture_bk }));
                materialArray.push(new THREE.MeshBasicMaterial( { map: texture_up }));
                materialArray.push(new THREE.MeshBasicMaterial( { map: texture_dn }));
                materialArray.push(new THREE.MeshBasicMaterial( { map: texture_rt }));
                materialArray.push(new THREE.MeshBasicMaterial( { map: texture_lf }));

                for (let i = 0; i < 6; i++)
                    materialArray[i].side = THREE.BackSide;
                let skyboxGeo = new THREE.BoxGeometry( 2000, 2000, 2000);
                let skybox = new THREE.Mesh( skyboxGeo, materialArray );
                scene.add( skybox );
                animate()
                function animate() {
                    renderer.render(scene,camera);
                    requestAnimationFrame(animate);
                }
                this.gData=gData;
                this.Graph=Graph;
            },
            handleSelect(item) {
                console.log(item);
                this.$bus.$emit('clickItem',item.id)
            },
            querySearch(queryString, cb) {
                let idList = this.idList;
                let results = queryString ? idList.filter(this.createFilter(queryString)) : idList;
                // 调用 callback 返回建议列表的数据
                cb(results);
            },
            createFilter(queryString) {
                return (item) => {
                    return (item.value.toLowerCase().indexOf(queryString.toLowerCase()) === 0);
                };
            },
            loadData(){
                request({
                    url:'/marvelId'
                }).then(res => {
                    this.idList = res.id
                    // console.log(res);
                })
            },
            filterGraphData(arr){
                if(arr.length===0){
                    this.selectAll=true
                    this.initGraph1Again(true)
                }
                this.selectAll=false
                arr=arr.map(function (value,index,array) {
                    return value.toLowerCase()
                })
                console.log(arr)
                let nodes=this.gData.nodes.filter(function (node) {
                    if(arr.includes(node.type)) return true;
                    else {
                        let aff=Object.keys(node.affiliation);
                        for(let item of aff){
                            return arr.includes(item)
                        }
                    }
                })
                let links=this.gData.links.filter(function (link) {
                    return nodes.includes(link.source)&&nodes.includes(link.target)
                })
                this.Graph.graphData({
                    nodes:nodes,
                    links:links
                })
            },
            initGraph1Again(value){
                if(value){
                    this.Graph.graphData(this.gData);
                    this.checkedList=[]
                }
            }
        }
    }
</script>
<style scoped>
    #3d-graph{
          width: 100%;
          height: 100%;
      }

    .query{
        position: fixed;
        left: 60px;
        top: 80px;
        z-index: 5;
    }

    /deep/.el-input__inner{
        background-color: rgba(0,0,0,0.1);
        border: rgba(255,255,255,0.3) 1px solid;
        color: white;
    }

    /deep/.el-checkbox{
        color: white;
    }

    /deep/.el-checkbox__input{
        /*display: none;*/
    }

    .check{
        margin-top: 8px;
    }
</style>