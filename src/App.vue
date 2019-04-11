<template>
  <div id="app" class="noselect app flex">
    <app-points-list class="column left"></app-points-list>
    <app-google-maps class="column right"></app-google-maps>
  </div>
</template>
<script>
import PointsList from "./components/PointsList.vue";
import GoogleMaps from "./components/GoogleMaps.vue";

export default {
  name: "app",
  components: {
    appPointsList: PointsList,
    appGoogleMaps: GoogleMaps
  }
};
</script>

<style>
body,
html {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
*,
*::before,
*::after {
  box-sizing: inherit;
}
.noselect {
  -webkit-touch-callout: none; /* iOS Safari */
  -webkit-user-select: none; /* Safari */
  -moz-user-select: none; /* Firefox */
  -ms-user-select: none; /* Internet Explorer/Edge */
  user-select: none; /* Non-prefixed version, currently supported by Chrome and Opera */
}
.app {
  height: 100vh;
  min-width: 500px;
}
.flex {
  display: flex;
  flex-flow: row wrap;
  flex: 1 0 auto;
}
.column {
  /* display: flex;
    flex-flow: column wrap;
    align-items: center;
    justify-content: center;       */
  padding: 10px;
}
.left {
  width: 25%;
  /* align-items: flex-end;     */
  background-color: gainsboro;
}
.right {
  width: 75%;
}
</style>
