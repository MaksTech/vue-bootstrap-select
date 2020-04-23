<template>
  <div
    v-on-clickaway="hideDropdown"
    @keyup.esc="onEscape"
    @keydown.up.prevent="typeAheadUp"
    @keydown.down.prevent="typeAheadDown"
    @keydown.enter.prevent="typeAheadSelect"
    class="dropdown bootstrap-select"
    :class="{
      disabled,
    }"
  >
    <button
      @click.prevent="toggle"
      type="button"
      class="dropdown-toggle form-control"
      :class="labelClass"
    >
      <span v-html="title"></span>
    </button>
    <div
      class="dropdown-menu"
      :class="{
        show,
      }"
    >
      <div v-show="searchable" class="bs-searchbox">
        <input
          :placeholder="labelSearchPlaceholder"
          class="form-control"
          :class="searchClass"
          type="text"
          v-model="searchValue"
          autofocus
        />
      </div>
      <ul class="dropdown-menu inner show">
        <li
          v-show="searchable && filteredOptions.length === 0"
          class="v-dropdown-item"
        >
          {{ labelNotFound }} "{{ searchValue }}"
        </li>
        <li
          v-if="showDefaultOption"
          class="dropdown-item disabled default-option"
        >
          {{ labelTitle }}
        </li>
        <li
          v-for="(option, index) in filteredOptions"
          :key="`v-select-${index}`"
          :class="{
            disabled: option[disabledProp],
          }"
        >
          <a
            class="dropdown-item"
            :class="{
              active: isSelectedOption(option, index),
              disabled: option[disabledProp],
            }"
            @click.prevent="onSelect(option, index)"
            v-html="getOptionLabel(option)"
          ></a>
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
import { mixin as clickaway } from 'vue-clickaway';

export default {
  name: 'VSelect',
  mixins: [clickaway],
  props: {
    disabled: {
      type: Boolean,
      default: false,
    },
    disabledProp: {
      type: String,
      default: 'disabled',
    },
    labelClass: {
      //
    },
    labelTitle: {
      type: String,
      default: 'Nothing selected',
    },
    labelNotFound: {
      type: String,
      default: 'No results matched',
    },
    labelSearchPlaceholder: {
      type: String,
      default: 'Search',
    },
    options: {
      type: Array,
      default: () => [],
    },
    searchable: {
      type: Boolean,
      default: false,
    },
    searchClass: {
      //
    },
    showDefaultOption: {
      type: Boolean,
      default: false,
    },
    textProp: {
      type: String,
      default: 'text',
    },
    value: {
      type: [Object, String, Number],
      default: null,
    },
    valueProp: {
      type: String,
      default: 'value',
    },
  },
  data() {
    return {
      show: false,
      selectedValue: null,
      searchValue: '',
      typeAheadPointer: -1,
    };
  },
  computed: {
    title() {
      return this.selectedValue
        ? this.getOptionLabel(this.selectedValue)
        : this.labelTitle;
    },
    filteredOptions() {
      if (this.searchable && this.searchValue.length > 0) {
        return this.options.filter(item => {
          if (typeof item === 'object') {
            return (
              item[this.textProp]
                .toLowerCase()
                .indexOf(this.searchValue.toLowerCase()) !== -1
            );
          } else {
            return (
              item.toLowerCase().indexOf(this.searchValue.toLowerCase()) !== -1
            );
          }
        });
      }
      return this.options;
    },
    reversedOptions() {
      return [...this.filteredOptions].reverse();
    },
    lastOptionIndex() {
      return this.filteredOptions.length - 1;
    },
  },
  watch: {
    value: {
      immediate: true,
      handler(newVal) {
        const index = this.options.findIndex(op =>
          this.isEqualOption(op, newVal),
        );
        this.onSelect(newVal, index);
      },
    },
  },
  methods: {
    onSelect(option, index) {
      if (option && !option[this.disabledProp]) {
        this.selectedValue = option;
        this.typeAheadPointer = index;
        this.hideDropdown();
        this.$emit('input', option, option[this.valueProp], index);
      } else if (option === null) {
        this.selectedValue = null;
      }
    },
    onEscape() {
      this.hideDropdown();
    },
    typeAheadUp() {
      if (!this.show) {
        this.show = true;
      }
      if (this.typeAheadPointer > 0) {
        const nextPointer = this.typeAheadPointer - 1;
        const option = this.filteredOptions[nextPointer];
        const isDisabled = option ? option[this.disabledProp] || false : false;
        if (!isDisabled) {
          this.typeAheadPointer--;
        } else {
          this.typeAheadPointer--;
          this.typeAheadUp();
        }
      } else {
        const nextEnabledOption = this.reversedOptions.findIndex(
          o => o[this.disabledProp] !== true,
        );
        this.typeAheadPointer = this.lastOptionIndex - nextEnabledOption;
      }
    },
    typeAheadDown() {
      if (!this.show) {
        this.show = true;
      }
      if (this.typeAheadPointer < this.lastOptionIndex) {
        const nextPointer = this.typeAheadPointer + 1;
        const option = this.filteredOptions[nextPointer];
        const isDisabled = option ? option[this.disabledProp] || false : false;
        if (!isDisabled) {
          this.typeAheadPointer++;
        } else {
          this.typeAheadPointer++;
          this.typeAheadDown();
        }
      } else {
        this.typeAheadPointer = this.filteredOptions.findIndex(
          o => o[this.disabledProp] !== true,
        );
      }
    },
    typeAheadSelect() {
      if (this.filteredOptions[this.typeAheadPointer]) {
        this.onSelect(
          this.filteredOptions[this.typeAheadPointer],
          this.typeAheadPointer,
        );
      }
    },
    hideDropdown() {
      this.show = false;
      this.searchValue = '';
    },
    getOptionLabel(option) {
      if (typeof option === 'object') {
        return option[this.textProp];
      }
      return option;
    },
    isSelectedOption(option, index) {
      if (this.typeAheadPointer === -1 && this.selectedValue) {
        return this.isEqualOption(option, this.selectedValue);
      }
      return this.typeAheadPointer === index;
    },
    isEqualOption(a, b) {
      if (a && b && typeof a === 'object' && typeof b === 'object') {
        return (
          a[this.textProp] === b[this.textProp] &&
          a[this.valueProp] === b[this.valueProp]
        );
      }
      return a === b;
    },
    toggle() {
      if (!this.disabled) {
        this.show = !this.show;
      }
    },
  },
};
</script>
