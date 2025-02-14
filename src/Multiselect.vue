<template>
  <div
    ref="multiselect"
    :tabindex="tabindex"
    :class="classList.container"
    :id="id"
    @focusin="activate"
    @focusout="deactivate"
    @keydown="handleKeydown"
    @focus="handleFocus"
  >
    <!-- Search -->
    <template v-if="mode !== 'tags' && searchable && !disabled">
      <input    
        :modelValue="search"
        :value="search"
        :class="classList.search"
        @input="handleSearchInput"
        ref="input"
      />
    </template>

    <!-- Tags (with search) -->
    <template v-if="mode == 'tags'">
      <div :class="classList.tags">

        <slot
          v-for="(option, i, key) in iv"
          name="tag"
          :option="option"
          :handleTagRemove="handleTagRemove"
          :disabled="disabled"
        >
          <span :class="classList.tag" :key="key">
            {{ option[label] }}
            <span
              v-if="!disabled"
              :class="classList.tagRemove"
              @mousedown.prevent="handleTagRemove(option, $event)"
            >
              <span :class="classList.tagRemoveIcon"></span>
            </span>
          </span>
        </slot>
    
        <div :class="classList.tagsSearchWrapper">
          <!-- Used for measuring search width -->
          <span :class="classList.tagsSearchCopy">{{ search }}</span>

          <!-- Actual search input -->
          <input    
            v-if="searchable && !disabled"
            :modelValue="search"
            :value="search"
            :class="classList.tagsSearch"
            @input="handleSearchInput"
            ref="input"
          />
        </div>
      </div>
    </template>

    <!-- Single label -->
    <template v-if="mode == 'single' && hasSelected && !search && iv">
      <slot name="singlelabel" :value="iv">
        <div :class="classList.singleLabel">
          {{ iv[label] }}
        </div>
      </slot>
    </template>

    <!-- Multiple label -->
    <template v-if="mode == 'multiple' && hasSelected && !search">
      <slot name="multiplelabel" :values="iv">
        <div :class="classList.multipleLabel">
          {{ multipleLabelText }}
        </div>
      </slot>
    </template>

    <!-- Placeholder -->
    <template v-if="placeholder && !hasSelected && !search">
      <slot name="placeholder">
        <div :class="classList.placeholder">
          {{ placeholder }}
        </div>
      </slot>
    </template>

    <!-- Spinner -->
    <slot v-if="busy" name="spinner">
      <span :class="classList.spinner"></span>
    </slot>

    <!-- Clear -->
    <slot v-if="hasSelected && !disabled && canClear && !busy" name="clear" :clear="clear">
      <span :class="classList.clear" @mousedown="clear"><span :class="classList.clearIcon"></span></span>
    </slot>

    <!-- Caret -->
    <slot v-if="caret" name="caret">
      <span :class="classList.caret" @click="handleCaretClick"></span>
    </slot>

    <!-- Options -->
    <div
      :class="classList.dropdown"
      tabindex="-1"
    >
      <slot name="beforelist" :options="fo"></slot>

      <ul :class="classList.options">
        <li
          v-for="(option, i, key) in fo"
          :class="classList.option(option)"
          :key="key"
          :data-pointed="isPointed(option)"
          @mouseenter="setPointer(option)"
          @click="handleOptionClick(option)"
        >
          <slot name="option" :option="option" :search="search">
            <span>{{ option[label] }}</span>
          </slot>
        </li>
      </ul>

      <slot v-if="noOptions" name="nooptions">
        <div :class="classList.noOptions" v-html="noOptionsText"></div>
      </slot>

      <slot v-if="noResults" name="noresults">
        <div :class="classList.noResults" v-html="noResultsText"></div>
      </slot>

      <slot name="afterlist" :options="fo"></slot>
    </div>

    <!-- Hacky input element to show HTML5 required warning -->
    <input v-if="required" :class="classList.fakeInput" tabindex="-1" :value="textValue" required/>
    
    <!-- Native input support -->
    <template v-if="nativeSupport">
      <input v-if="mode == 'single'" type="hidden" :name="name" :value="plainValue !== undefined ? plainValue : ''" />
      <template v-else>
        <input v-for="(v, i) in plainValue" type="hidden" :name="`${name}[]`" :value="v" :key="i" />
      </template>
    </template>

    <!-- Create height for empty input -->
    <div :class="classList.spacer"></div>

  </div>
</template>

<script>
  import useData from './composables/useData'
  import useValue from './composables/useValue'
  import useSearch from './composables/useSearch'
  import usePointer from './composables/usePointer'
  import useOptions from './composables/useOptions'
  import usePointerAction from './composables/usePointerAction'
  import useDropdown from './composables/useDropdown'
  import useMultiselect from './composables/useMultiselect'
  import useKeyboard from './composables/useKeyboard' 
  import useClasses from './composables/useClasses' 

  export default {
    name: 'Multiselect',
    emits: [
      'open', 'close', 'select', 'deselect', 
      'input', 'search-change', 'tag', 'update:modelValue',
      'change', 'clear'
    ],
    props: {
      value: {
        required: false,
      },
      modelValue: {
        required: false,
      },
      options: {
        type: [Array, Object, Function],
        required: false,
        default: () => ([])
      },
      id: {
        type: [String, Number],
        required: false,
      },
      name: {
        type: [String, Number],
        required: false,
        default: 'multiselect',
      },
      disabled: {
        type: Boolean,
        required: false,
        default: false,
      },
      label: {
        type: String,
        required: false,
        default: 'label',
      },
      trackBy: {
        type: String,
        required: false,
        default: 'label',
      },
      valueProp: {
        type: String,
        required: false,
        default: 'value',
      },
      placeholder: {
        type: String,
        required: false,
        default: null,
      },
      mode: {
        type: String,
        required: false,
        default: 'single', // single|multiple|tags
      },
      searchable: {
        type: Boolean,
        required: false,
        default: false,
      },
      limit: {
        type: Number,
        required: false,
        default: -1,
      },
      hideSelected: {
        type: Boolean,
        required: false,
        default: true,
      },
      createTag: {
        type: Boolean,
        required: false,
        default: false,
      },
      appendNewTag: {
        type: Boolean,
        required: false,
        default: true,
      },
      caret: {
        type: Boolean,
        required: false,
        default: true,
      },
      loading: {
        type: Boolean,
        required: false,
        default: false,
      },
      noOptionsText: {
        type: String,
        required: false,
        default: 'The list is empty',
      },
      noResultsText: {
        type: String,
        required: false,
        default: 'No results found',
      },
      multipleLabel: {
        type: Function,
        required: false,
      },
      object: {
        type: Boolean,
        required: false,
        default: false,
      },
      delay: {
        type: Number,
        required: false,
        default: -1,
      },
      minChars: {
        type: Number,
        required: false,
        default: 0,
      },
      resolveOnLoad: {
        type: Boolean,
        required: false,
        default: true,
      },
      filterResults: {
        type: Boolean,
        required: false,
        default: true,
      },
      clearOnSearch: {
        type: Boolean,
        required: false,
        default: false,
      },
      clearOnSelect: {
        type: Boolean,
        required: false,
        default: true,
      },
      canDeselect: {
        type: Boolean,
        required: false,
        default: true,
      },
      canClear: {
        type: Boolean,
        required: false,
        default: true,
      },
      max: {
        type: Number,
        required: false,
        default: -1,
      },
      showOptions: {
        type: Boolean,
        required: false,
        default: true,
      },
      addTagOn: {
        type: Array,
        required: false,
        default: () => (['enter']),
      },
      required: {
        type: Boolean,
        required: false,
        default: false,
      },
      openDirection: {
        type: String,
        required: false,
        default: 'bottom',
      },
      nativeSupport: {
        type: Boolean,
        required: false,
        default: false,
      },
      classes: {
        type: Object,
        required: false,
        default: () => ({})
      },
      strict: {
        type: Boolean,
        required: false,
        default: true,
      },
      closeOnSelect: {
        type: Boolean,
        required: false,
        default: true,
      },
    },
    setup(props, context)
    { 
      const value = useValue(props, context)
      const pointer = usePointer(props, context)
      const dropdown = useDropdown(props, context)
      const search = useSearch(props, context)

      const data = useData(props, context, {
        iv: value.iv,
      })

      const multiselect = useMultiselect(props, context, {
        input: search.input,
        open: dropdown.open,
        close: dropdown.close,
        clearSearch: search.clearSearch,
      })

      const options = useOptions(props, context, {
        ev: value.ev,
        iv: value.iv,
        search: search.search,
        clearSearch: search.clearSearch,
        update: data.update,
        pointer: pointer.pointer,
        blur: multiselect.blur,
        deactivate: multiselect.deactivate,
      })

      const pointerAction = usePointerAction(props, context, {
        fo: options.fo,
        handleOptionClick: options.handleOptionClick,
        search: search.search,
        pointer: pointer.pointer,
        multiselect: multiselect.multiselect,
      })

      const keyboard = useKeyboard(props, context, {
        iv: value.iv,
        update: data.update,
        search: search.search,
        setPointer: pointerAction.setPointer,
        selectPointer: pointerAction.selectPointer,
        backwardPointer: pointerAction.backwardPointer,
        forwardPointer: pointerAction.forwardPointer,
        blur: multiselect.blur,
        fo: options.fo,
      })

      const classes = useClasses(props, context, {
        isOpen: dropdown.isOpen,
        isPointed: pointerAction.isPointed,
        isSelected: options.isSelected,
        isDisabled: options.isDisabled,
        isActive: multiselect.isActive,
      })

      return {
        ...value,
        ...dropdown,
        ...multiselect,
        ...pointer,
        ...data,
        ...search,
        ...options,
        ...pointerAction,
        ...keyboard,
        ...classes,
      }
    }
  }
</script>