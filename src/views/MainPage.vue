<template>
    <div class="header-style d-flex flex-row" :style="{height:sideMenuProps.header_height}">
        <div class="d-flex flex-row justify-content-between w-100">
            <toggle-button
                class="toggle-button-style enable-icon"
                :target_id="'#'+sideMenuProps.target_canvas_id"
            />
            <div class="d-flex flex-row">
                <div class="account-info enable-icon"
                    @click="$router.push('profile')"
                    v-if="STATE.status.loggedIn"
                >
                    <span v-if="STATE.user" class="account-name text-white">{{STATE.user.username}}</span>
                    <BootstrapIcon
                        icon="person-circle"
                        size="2x"
                        variant="light"
                        />           
                </div>
                <div class="account-login-logout d-flex flex-row"
                    v-if="STATE.status.logout"
                >
                    <BootstrapIcon
                        icon="person-plus-fill"
                        size="2x"
                        variant="light"
                        class="enable-icon"
                        @click="$router.push('signup')"
                        style="margin-right: 15px;"
                        />
                    <BootstrapIcon
                        icon="person-check"
                        size="2x"
                        variant="light"
                        class="enable-icon"
                        @click="$router.push('signin')"
                        />
                </div>
            </div>
        </div>
    </div>
    <div class="info-div text-light">
        <div v-if="targetProject" class="d-flex felx-row justify-content-between">
            <div>Проект:</div>
            <div>{{targetProject.name}}</div>
        </div>
        <hr style="margin: 5px 0;">
        <div>Координаты курсора:</div>
        <div class="d-flex felx-row justify-content-between mouse-coord-style">
            <div>lat:</div>
            <div>{{mouseCoord[0]!=0?mouseCoord[0]:'???'}}</div>
        </div>
        <div class="d-flex felx-row justify-content-between mouse-coord-style">
            <div>lon:</div>
            <div>{{mouseCoord[1]!=0?mouseCoord[1]:'???'}}</div>
        </div>
        <div class="d-flex felx-row justify-content-between mouse-coord-style">
            <div>alt:</div>
            <div>{{mouseCoord[2]?mouseCoord[2]:'???'}}</div>
        </div>
    </div>
    <div class="all-window-height w-100">
        <yandex-map-component
            v-model:mouseCoord="mouseCoord"
            v-model:zoom="map_parameters.zoom"
            v-model:center="map_parameters.center"
            :flying="flying"
            :objects="objects"
            @deployFlyChange="deployFlyChange"
            @clickImage="clickFlyImageEvent"
            @clickObject="clickObjectEvent"
            @editFly="editFlyEvent"
            @updateFlying="fetchFlying(target_project_id)"
            @updateObjects="fetchObjects(target_project_id)"
            @rightClickMap="createNewObjectEvent"
            @rightClickObject="addDataToObjectEvent"
        />
    </div>
    <projects-menu
        :canvas_props="sideMenuProps"
        :projects="searcherProjects"
        @targetUpdate="updateTargetProject"
        @addEvent="addProjectEvent"
        @deleteProjectEvent="deleteProjectEventWithSideBar"
        v-model="searchQuery"
    />
    <fly-object-menu
        :objects="objects"
        :flying="flying"
        @deployFlyChange="deployFlyChange"
        @addFlyEvent="addFlyEvent"
        @centerMapOnObject="centerMapOnObject"
        v-if="viewFlyObjectMenu&&creatingObject==null"
    />
    <div
        v-if="creatingObject" 
        class="add-object-info-style text-white d-flex justify-content-center align-items-center"
    >
        <div>Нажмите правую кнопку мыши для сохранения объекта на карте</div>
    </div>
    <loading
        v-model="loading"
    />
    <fly-image-component
        v-model:show="flyImageShow"
        :flying="flying"
        @createObject="setCreatingObjectEvent"
        @updateActivePhoto="updateActivePhotoEvent($event); centerMapOnObject(activeFlyPhoto.photo.coords)"
    />
    <object-image-component
        v-model:show="objectImageShow"
        :object="activeObject"
        @deleteObjectElemEvent="deleteObjectElemEvent"
    />
    <add-project-window
        v-model:show="addProjectShow"
        @updateProjects="fetchProjects(); fetchFlying($event); fetchObjects($event); target_project_id=$event"
    />
    <add-fly-window
        v-model:show="addFlyShow"
        v-model:loading="creatingFly"
        :project_id="target_project_id"
        :editFly="editFly"
    />
    <delete-project-window
        v-model:show="deleteProjectShow"
        :project="deletingProject"
        @updateProjects="fetchProjects"
    />
</template>

<script>
import YandexMapComponent from '@/components/yandexMapComponent/YandexMapComponent.vue'
import ToggleButton from '@/components/UI/ToggleButton.vue'
import ProjectsMenu from '@/components/projectMenu/ProjectsMenu.vue'
import Loading from '@/components/Loading.vue'
import FlyObjectMenu from '@/components/flyObjectMenu/FlyObjectMenu.vue'
import FlyImageComponent from '../components/FlyImageComponent.vue'
import ObjectImageComponent from '../components/ObjectImageComponent.vue'

import { mapGetters } from 'vuex'
import AddProjectWindow from '../components/AddProjectWindow.vue'
import AddFlyWindow from '../components/addFlyComponent/AddFlyWindow.vue'

import {useProjectsApi} from '@/views/MainPage/projects.js'
import {useFlyingApi} from '@/views/MainPage/flying.js'
import {useObjectsApi} from '@/views/MainPage/objects.js'
import DeleteProjectWindow from '../components/DeleteProjectWindow.vue'

export default {
    setup(){
        const apiProject = useProjectsApi()
        const flyingApi = useFlyingApi()
        const objectApi = useObjectsApi()
        return {...apiProject, ...flyingApi, ...objectApi}
    },
    components:{
        YandexMapComponent,
        ToggleButton,
        ProjectsMenu,
        Loading,
        FlyObjectMenu,
        FlyImageComponent,
        ObjectImageComponent,
        AddProjectWindow,
        AddFlyWindow,
        DeleteProjectWindow,
    },
    data(){
        return {
            sideMenuProps:{
                target_canvas_id: "sideMenu",
                header_height: '50px'
            },
            map_parameters:{
                center: [55.737722, 37.732367],
                zoom: 6,
            },
            mouseCoord: [0,0,0]
        }
    },

    computed:{
        viewFlyObjectMenu(){ //флаг, сообщает о необходимости показывать и не показывать списки полетов и объектов
            return this.target_project_id
        },
        loading(){ //статус загрузки
            return this.loadingFly || this.loadingObject || this.loadingProject || this.creatingFly
        },
        ...mapGetters('account',['STATE']),
    },
    methods:{
        updateTargetProject(id){ // при выборе проекта меняется id активного проекта, подтягивается данные полетов и объектов для проекта
            this.target_project_id = id
            this.fetchFlying(id)
            this.fetchObjects(id)
            this.closeSideMenu()
        },
        addProjectEvent(){ // вызов нового окна создания нового проекта
            this.addProjectShow = true
            this.closeSideMenu()
        },
        deleteProjectEventWithSideBar(e){
            this.closeSideMenu()
            this.deleteProjectEvent(e)
        },
        closeSideMenu(){ // закрыть боковое меню с проектами 
            var link = document.getElementById('#'+this.sideMenuProps.target_canvas_id+'_button');
            var event = new CustomEvent("click")
            link.dispatchEvent(event)
        },

        deployFlyChange(cmd){ // устанавливает свойство deploy в нужное состояние для полета с id (можно перенести в файл flying.js, если поправить)
            let fly = this.getFlyById(cmd.id);
            fly.deployed = cmd.deployed;
            if(fly.photos.length!=0){
                this.centerMapOnObject(fly.photos[0].coords)
            }
        },
        centerMapOnObject(coords){ // устанавливает экран пользователя по указанным координатам
            this.map_parameters.center = coords
        },
        centeringMap(){ // центрирование и зумирование карты по данным объектов и полетов
            let max_min_lat = [0,0]
            let max_min_lon = [0,0]
            if (this.objects.length!=0){
                max_min_lat[0] = this.objects[0].coords[0]
                max_min_lat[1] = this.objects[0].coords[0]

                max_min_lon[0] = this.objects[0].coords[1]
                max_min_lon[1] = this.objects[0].coords[1]
            }
            else{
                return
            }
            this.objects.forEach((object)=>{
                if (object.coords[0]>max_min_lat[0]){
                    max_min_lat[0]=object.coords[0]
                }
                else if(object.coords[0]<max_min_lat[1]){
                    max_min_lat[1] = object.coords[0]
                }
                
                if (object.coords[1]>max_min_lon[0]){
                    max_min_lon[0] = object.coords[1]
                }
                else if(object.coords[1]<max_min_lon[1]){
                    max_min_lon[1] = object.coords[1]
                }
            })
            this.map_parameters.center = [(max_min_lat[0]+max_min_lat[1])/2, (max_min_lon[0]+max_min_lon[1])/2]
            this.map_parameters.zoom = 6
        },
        async createNewObjectEvent(){ //создание нового объекта по клику на карту
            if(this.creatingObject){
                this.createObject(this.target_project_id, this.mouseCoord, this.STATE.token.token)
                this.setFlyingPhotoActive(this.creatingObject.fly_id, this.creatingObject.photo_id)
                this.flyImageShow=true
            }
        },
        async addDataToObjectEvent(id){ // добавление наблюдения к объекту
            if(this.creatingObject){
                this.createObject(this.target_project_id, this.mouseCoord, this.STATE.token.token, id)
                this.setFlyingPhotoActive(this.creatingObject.fly_id, this.creatingObject.photo_id)
                this.flyImageShow=true
            }
        },
        checkLoadFlyImage(){
            if(!this.loadingFly && !this.loadingObject){
                this.centeringMap()
            }
            else{
                setTimeout(this.checkLoadFlyImage, 100)
            }
        }
    },
    watch:{
        objectImageShow(new_v){ // после закрытия объекта все объекты становятся неактивными
            if(!new_v){
                this.setAllObjectsInactive()
            }
        },
        target_project_id(new_v){ // после завершения загрузки отцентрировать карту
            if(new_v){
                setTimeout(this.checkLoadFlyImage,100)
            }
        },
        addFlyShow(new_v){ // после закрытия окна создания полета подгрузить новые полеты
            if(!new_v){
                this.editFly = null
                this.fetchFlying(this.target_project_id)
            }
        },
        creatingObject(new_v){
            if(new_v==null){
                this.fetchObjects(this.target_project_id)
            }
        }
    },
    mounted(){ // загружаются проекты
        this.fetchProjects()
    }
}
</script>

<style scoped>
.header-style{
    width: 100%;
    background: #3f3f3fca;
    border-bottom: 1px solid white;
    position: fixed;
    z-index: 1;
}
.info-div{
    position: fixed;
    width: 250px;
    background-color: rgba(76, 74, 74, 0.781);
    border-radius: 3px;
    right: 0;
    top:52px;
    z-index: 3;
    padding: 0 5px 0 5px;
}
.mouse-coord-style{
    font-size: 0.85rem;
    margin-left: 7px;
}
.account-login-logout{
    width: 100px;
    justify-content: space-between;
    align-self: center;
    padding: 0 10px;
}
.account-info{
    align-self: center;
    padding: 0 10px;
    margin-right: 10px;
}
.account-name{
    margin-right: 10px;
}
.toggle-button-style{
    align-self: center;
    margin-left: 20px;
}
.add-object-info-style{
    position: absolute;
    bottom: 0;
    height: 50px;
    width: 100%;
    background-color: rgba(76, 74, 74, 0.781);
    border-top: 1px solid gray;
    font-weight: 600;
}
</style>