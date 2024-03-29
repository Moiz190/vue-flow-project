<template>
  <div
    class="dropdown"
    id="dropdownMenu"
    :style="{
      top: `${top}px`,
      left: `${left}px`,
      transform: isMenuVisible ? 'scale(1)' : 'scale(0)',
      transformOrigin: isMenuVisible ? 'top left' : '',
      transition: isMenuVisible ? 'transform 200ms ease-in-out' : 'none',
    }"
  >
    <div class="dropdownItem">Add a Condition</div>
  </div>
</template>
<script setup lang="ts">
interface IProps {
  left: number;
  top: number;
  isMenuVisible: boolean;
}
withDefaults(defineProps<IProps>(), {
  left: 0,
  top: 0,
  isMenuVisible: false,
});
</script>
<style scoped>
#dropdownMenu {
  position: fixed;
  z-index: 100000;
  width: 10rem;
  background: #1b1a1a;
  border-radius: 5px;
}
.dropdown .dropdownItem {
  padding: 0.5rem 0.7rem;
  font-size: 0.9rem;
  color: #eee;
  cursor: pointer;
  border-radius: inherit;
}
.dropdown .dropdownItem:hover {
  background: #343434;
}
</style>
