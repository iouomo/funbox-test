<template>
  <div class="container">
    <div class="list">
      <input
        class="input"
        v-model="newPointText"
        placeholder="Новая точка маршрута"
        @keyup.enter="addNewPoint"
      >
      <ul class="points">
        <li
          v-for="(point, index) in sortedPoints"
          :key="index"
          :ref="'point_' + index"
          draggable="true"
          @dragstart="dragstart(point, $event)"
          @dragend="dragend($event)"
          @dragenter="dragenter(point, $event)"
          class="point"
        >
          {{ point.title }}
          <button @click="removeItem(index, point.id)" class="button">Удалить</button>
        </li>
      </ul>
    </div>
  </div>
</template>
<script>
import { eventBus } from "../../src/main";
export default {
  methods: {
    // Создает новую точку маршрута и добавляет ее в массив точек. Точки с маркерами связаны через свойство id.
    addNewPoint() {
      if (this.newPointText === "") {
        alert("Не указано наименование для новой точки!");
        return;
      }
      let data = {
        id: this.nextPoinId++,
        title: this.newPointText
      };
      this.points.push(data);
      // После добавления точки, посылаем событие на создание маркера на карте
      eventBus.$emit('addNewMarker', data);
      this.newPointText = "";
    },
    // Удаляем указанную точку из массива точек
    removeItem(index, id) {
      this.points.splice(index, 1);
      // Обновляем индекс следующей точки
      if (this.points.length === 0) {
        this.nextPoinId = 0;
      } else {
        let maxId = Math.max(...this.points.map((elem) => elem.id, 0));
        this.nextPoinId = ++maxId;
      }
      // После удаления точки, посылаем событие на удаление маркера на карте
      eventBus.$emit('removeMarker', id);
    },
    //События по перетаскиванию в списке точек
    dragstart(item, event) {
      this.draggedElement = item; // Запоминаем перетаскиваемую точку
      this.draggedStartIndex = item.id; // Запоминаем id перетаскиваемой точки
      event.target.style.opacity = 0.5; // Для визуального представления уменьшаем прозрачномть перетаскиваемой точки
    },
    dragend(event) {
      event.target.style.opacity = 1;
      // Если положение точки изменилось, отправляем событие на перерисовку маршрута на карте   
      if (this.draggedStartIndex !== this.draggedElement.id) {
        eventBus.$emit('realignMarkers');  
      }
    },
    dragenter(item, event) {
      // Меняем идентифиакторы у перетаскиваемой точки и точки, на место которой встала перетаскиваемая точка.
      let tempIndex = item.id;      
      item.id = this.draggedElement.id; 
      // Посылаем событие на изменение идентифиакторов у маркеров.
      eventBus.$emit('swapMarkers', {
        from: tempIndex,
        to: item.id
      });

      this.draggedElement.id = tempIndex;
    }
  },
  computed: {
    // Отображаем отсортированный список точек в порядке возрастания их идентификаторов.
    sortedPoints() {
      return this.points.sort((a, b) => a.id - b.id);
    },  
  },
  data() {
    return {
      newPointText: "",
      nextPoinId: 0,
      draggedElement: undefined,
      draggedStartIndex: 0,
      points: [],
    };
  },
  watch: {}
};
</script>
<style scoped>
.container {
  display: flex;
  flex-flow: column wrap;
  flex: 1 0 auto;
  justify-content: center;
  align-items: flex-end;
}
.list {
  width: 100%;
  height: 75%;
  display: flex;
  flex-direction: column;
  flex-wrap: wrap;
  justify-content: flex-start;
  align-content: stretch;
}
.input {
  width: 100%;
}
.points {
  display: flex;
  flex-direction: column;
  flex-wrap: wrap;
  justify-content: space-between;
  align-items: flex-end;
  margin-left: 0;
  padding-left: 0;
  list-style-type: none;
}
.point {
  margin: 10px;
}
.point:hover {
  cursor: pointer;
}
.button{
  cursor: pointer;
}
</style>


