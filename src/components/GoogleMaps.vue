 <template>
  <div class="container">
    <div id="map" class="maps" ref="googleMaps"></div>
  </div>
</template>
<script>
// Шина событий.
import { eventBus } from "../../src/main";
export default {
  created() {
    // Подписки на события eventBus, которые поступают от списка точек.
    eventBus.$on("addNewMarker", data => {
      if (this.map !== undefined) {
        this.addNewMarker(data);
      }
    }),
      eventBus.$on("removeMarker", id => {
        if (this.map !== undefined) {
          this.removeMarker(id);
        }
      }),
      eventBus.$on("swapMarkers", data => {
        if (this.map !== undefined) {
          this.swapMarkers(data);
        }
      });
    eventBus.$on("realignMarkers", () => {
      if (this.map !== undefined) {
        this.realignMarkers();
      }
    });
  },
  async mounted() {
    // Загружаем карту Google асинхронно.
    await this.loadGoogleMaps();
  },
  methods: {
    async loadGoogleMaps() {
      let tag = document.createElement("script");
      tag.type = "text/javascript";
      tag.src = `${this.url}?key=${this.key}`;
      tag.onload = () => this.initMap();
      document.body.appendChild(tag);
    },
    // Пытаемся получить разрешение на определение координат пользователя.
    initUserCoordinates() {
      if ("geolocation" in navigator) {
        navigator.geolocation.getCurrentPosition(location => {
          this.lat = location.coords.latitude;
          this.lng = location.coords.longitude;
        });
      }
    },
    // Инициализируем карту после загрузки скрипта от Google.
    initMap() {
      this.initUserCoordinates();

      let element = this.$refs.googleMaps;
      let options = {
        zoom: 15,
        center: new google.maps.LatLng(this.lat, this.lng)
      };
      this.map = new google.maps.Map(element, options);
      this.infoWindow = new google.maps.InfoWindow({
        content: "",
        map: this.map
      });
    },

    // MARKERS
    //

    // Добавляем новый маркер на карту и запоминаем его в массив маркеров. 
    addNewMarker(data) {
      let marker = this.newMarker(data);
      this.markers.push(marker);
      // Строим маршрут до нового маркера. 
      this.addNewPolyline(marker);
    },
    // Конструктор нового маркера.
    newMarker(props) {
      let latlng = this.markerLatLng();

      let marker = new google.maps.Marker({
        position: latlng,
        map: this.map,
        title: props.title,
        draggable: true,
        animation: google.maps.Animation.DROP
      });

      marker.id = props.id;

      // Подписка на событие перетаскивания маркера.
      marker.addListener("drag", event => {
        marker.setPosition(event.latLng); // При перетаскивании, изменяем координаты маркера.

        // При перетаскивании маркера, изменяем координаты маршрутов, которые связаны с перетаскиваемым маркером.
        let polylines = this.connectedPolylines(marker.id);
        polylines.forEach(elem => {
          elem.setPath([elem.topA.position, elem.topB.position]);
        });
      });
      // Подписка на событие клика на маркере и появления балуна с названием макера.
      marker.addListener("click", () => {
        this.infoWindow.setContent(props.title);
        this.infoWindow.open(this.map, marker);
      });

      return marker;
    },
    // Получаем координаты для маркера, из центра карты.
    markerLatLng() {
      let coord = this.map.getCenter();
      return new google.maps.LatLng(coord.lat(), coord.lng());
    },
    // Удаление маркера, по его идентификатору.
    removeMarker(id) {
      // Найдем маркер. 
      let currentMarker = this.findMarkerById(id);
      if (currentMarker === undefined) {
        return;
      }
      // Удалим найденный маркер с карты.
      currentMarker.setMap(null); 
      // Удалим маршруты с карты, связанные с удаляемым маркером.
      this.removeLines(id);

      // Перезапишем массив с маркерами, удалив из него маркер с указанным идентификатором.
      this.markers = this.markers.filter(elem => elem.id !== id);
      // Соединим соседние маркеры.
      this.lineNeighbourMarkers(id); 
    },
    // Поиск маркера по переданному идентификатору.
    findMarkerById(id) {
      return this.markers.find(elem => elem.id === id);
    },
    // Поиск следующего маркера по переданному идентификатору.
    nextMarker(id) {
      return this.markers.find(elem => elem.id > id);
    },
    // Поиск предыдущего маркера по переданному идентификатору.
    prevMarker(id) {
      let prevMarker = undefined;
      let tempArray = this.markers.filter(elem => elem.id < id);
      let length = tempArray.length;
      if (length > 0) {
        tempArray.sort((a, b) => a.id - b.id);
        prevMarker = tempArray[length - 1];
      }
      return prevMarker;
    },
    // Изменяем идентификаторы у маркеров, в процессе перетаскивания точек.
    swapMarkers(data) {
      let markerA = this.findMarkerById(data.from);
      let markerB = this.findMarkerById(data.to);
      markerA.id = data.to;
      markerB.id = data.from;
    },
    // Перестраиваем маршрут.
    realignMarkers() {
      this.lines.forEach(elem => {
        elem.setMap(null);
      });
      this.lines = [];

      this.markers.forEach(elem => {
        this.addNewPolyline(elem);
      });
    },
    
    // LINES
    //

    //Создаем новый маршрут, до указанного маркера.
    addNewPolyline(marker) {
      // Поищем предыдущий маркер, от которого будем строить маршрут.
      let prevMarker = this.prevMarker(marker.id);
      
      //Если предыдущий маркер не нашли, значит переданный маркер один и маршрут строить не надо.
      if (prevMarker === undefined) {
        return;
      }
      // Создаем новый маршрут и запоминаем его.
      let polyline = this.newPolyline(prevMarker, marker);
      this.lines.push(polyline);
    },
    // Конструктор нового маршрута.
    newPolyline(prevMarker, currMarker) {
      let polyline = new google.maps.Polyline({
        path: [prevMarker.position, currMarker.position],
        strokeColor: "#FF0000",
        strokeOpacity: 1.0,
        strokeWeight: 2,
        map: this.map
      });
      // Запоминаем маркеры, соединенные данным маршрутом. Это поможет нам, при удалении маркеров.
      polyline.topA = prevMarker;
      polyline.topB = currMarker;
      return polyline;
    },
    // Поиск маршрутов, с которыми связан маркер, с указанным идентификатором.
    connectedPolylines(id) {
      return this.lines.filter(
        elem => elem.topA.id === id || elem.topB.id === id
      );
    },
    // Удаление маршрутов, с которыми связан маркер, с указанным идентификатором. 
    removeLines(id) {
      // Найдем маршруты, с которыми связан маркер, с переданным идентификатором. 
      let polylines = this.connectedPolylines(id);
      // Для каждого маршрута очистим его представление на карте
      polylines.forEach(elem => {
        elem.setMap(null);
      });
      // Обновим массив с маршрутами, исключив из него найденные маршруты.
      this.lines = this.lines.filter(
        elem => elem.topA.id !== id && elem.topB.id !== id
      );
    },
    // Строим новый маршрут между маркерами, которые находятся по соседству с маркером, с указанным идентифиаткором. 
    lineNeighbourMarkers(id) {
      // Поищем соседние маркеры.
      let prevMarker = this.prevMarker(id);
      let nextMarker = this.nextMarker(id);
      
      // Если не нашли хотя бы одного из соседей, то и строить маршрут не имеет смысла.
      if (prevMarker === undefined || nextMarker === undefined) {
        return;
      }
      // Строим новый маршрут и запоминаем его в массив маршрутов.
      let polyline = this.newPolyline(prevMarker, nextMarker);
      this.lines.push(polyline);
    }
  },
  computed: {},
  data() {
    return {
      map: undefined,
      url: "https://maps.googleapis.com/maps/api/js",
      key: "AIzaSyBCdM0HJU2kUJcfsStsGCqM0KPyn0jdZ4Q",
      // По умолчанию, используем координаты MSK
      lat: 55.75222,
      lng: 37.61556,
      markers: [],
      lines: [],
      infoWindow: undefined
    };
  },
  watch: {}
};
</script>
<style scoped>
.container {
  display: flex;
  flex-flow: row wrap;
  flex: 1 0 auto;
  justify-content: flex-start;
  align-items: center;
}
.maps {
  width: 75%;
  height: 75%;
}
</style>


