<template>
  <div v-if="options" class="input-container" id="scrollingList">
    <!-- Dropdown Input -->
    <input
      class="new-tag rounded-left form-control"
      @focus="showOptions()"
      @blur="exit()"
      @keydown.tab.prevent="keyMonitor"
      v-model="searchFilter"
      :disabled="disabled"
      :placeholder="[[placeholderText]]"
      @keydown.enter="addTagTyping()"
      @keydown.delete.stop="removeLastTag"
      :style="widthLength"
      @keyup="autoWidth"
      :id="'feedbackCardInput'+id"
    />
    <!-- Dropdown Menu -->
    <div class="dropdown-content rounded-bottom" v-show="optionsShown" ref="options">
      <div
        class="dropdown-item pointer caption p-2"
        @mouseover="highlighted(index)"
        @mousedown="selectOption(option)"
        v-for="(option, index) in filteredOptions"
        :key="index"
        :class="{'highlightedOption':index === currentItem}"
        @keydown="scrollArrows()"
      >{{ option.name || option.id|| '-' }}</div>
    </div>
  </div>
</template>
<script>
import { mapGetters, mapActions } from "vuex";
import VueScroll from "vue-scrollto";

/**
 * Last updated: 12/06/2019
 * Comments: added beforeDestroy() 
 * updated by: Gabriel Torres
 */

export default {
  components: {
    VueScrollTo
  },
  props: {
    name: {
      type: String,
      required: false,
      default: "dropdown",
      note: "Input name"
    },
    options: {
      type: Array,
      required: true,
      default: [],
      note: "Options of dropdown. An array of options with id and name"
    },
    placeholder: {
      type: String,
      required: false,
      default: "Please select an option",
      note: "Placeholder of dropdown"
    },
    disabled: {
      type: Boolean,
      required: false,
      default: false,
      note: "Disable the dropdown"
    },
    maxItem: {
      type: Number,
      required: false,
      default: 6,
      note: "Max items showing"
    },
    tagsArray: {
      type: Array
    },
    id: {
      type: Number
    },
    updated: {
      type: Array
    }
  },
  data() {
    return {
      //selected takes in one of the objects from the options array (options = availableTagsList, but each string in availableTagsList is an object)
      selected: {},
      //determines if the dropdown is open or not
      optionsShown: false,
      //input typing field
      searchFilter: "",
      //positon of where you are in the dropdown (affected when using arrow keys or mouseover when the dropdown is open)
      currentItem: null,
      //width of the text box, changes as the placeholder changes or the text field
      widthNum: 70
    };
  },
  created() {
    this.$emit("selected", this.selected);
  },
  mounted() {
    document.addEventListener("keyup", this.scrollArrows);
  },
  //This is required to handle the error "children of undefined" encountered in one of the MessageCenterNewConver component, since 
  //it also uses a addEventListener()
  beforeDestroy() {
        document.removeEventListener('keyup', this.scrollArrows);
    },
  computed: {
    //widthLength is what determines if the text box width needs to get bigger
    widthLength: function() {
      //text box width expands based on how many characters are in the text field
      if (this.searchFilter.length > 6) {
        //width is defaulted to 70px btw
        this.widthNum = this.searchFilter.length * 11;
        //inline style bc I needed to use a variable
        return { width: this.widthNum + "px" };
      }
    },
    //as you type the dropdown will change based on what you type
    filteredOptions() {
      //all the tags in the filtered array is displayed in the drop down menu
      const filtered = [];
      //this checks the searchfilter and finds all the tags that start with the same letter as the the text you type
      const regOption = new RegExp("^(" + this.searchFilter + ")");
      for (const option of this.options) {
        if (
          option.name.match(regOption) &&
          //if the tag youre typing isn't inside CARD tagsarray then push it to the filtered array
          !this.tagsArray.includes(option.name)
        ) {
          filtered.push(option);
        }
      }
      //sorts it alphabetically
      filtered.sort((a, b) => (a.name > b.name ? 1 : -1));
      return filtered;
    },
    currTag: function() {
      //checks to see the text box chnaged (probably can get rid off)
      return this.searchFilter;
    },
    //makes the placeholdder text the same as whatever index/tag you're on in the dropdown
    placeholderText: function() {
      if (
        this.optionsShown === true &&
        this.filteredOptions[this.currentItem] !== undefined
      ) {
        return this.filteredOptions[this.currentItem].name;
      } else {
        return "Add Tag";
      }
    }
  },
  watch: {
    searchFilter() {
      if (this.filteredOptions.length === 0) {
        this.selected = {};
        this.currentItem = 0;
      } else {
        this.selected = this.filteredOptions[0];
        this.currentItem = 0;
      }
      this.$emit("filter", this.searchFilter);
    }
  },
  methods: {
    //vue x actions
    ...mapActions({
      addNewTag: "addNewTag",
      removeTag: "removeTag",
      setError: "setError"
    }),
    //checks to see that the tags is a duplicate
    validateTag(tag) {
      const newArr = this.tagsArray.map(element => {
        return element.toLowerCase();
      });
      return !newArr.includes(tag.toLowerCase()) && tag.length < 70;
    },
    //removes the last tag binded to backspace/delete key on keyboard
    removeLastTag() {
      if (this.currTag === "" || this.currTag === "") {
        const tags = this.tagsArray;
        const lastTag = tags[tags.length - 1];
        tags.pop();
        axios
          .delete(`/api/tags/feedback/${this.id}/${lastTag}`)
          .then(response => {
            this.removeTag(lastTag);
            this.tagsArray = tags;
          })
          .catch(error => {
            this.setError("There was a problem removing the tag. Please try again.");
          });
      }
    },
    autoWidth() {
      if (this.filteredOptions.includes(this.filteredOptions[0]) === true) {
        this.optionsShown = true;
      } else if (
        this.filteredOptions.includes(this.filteredOptions[0]) === false
      ) {
        this.optionsShown = false;
      }
    },
    addTagTyping() {
      const newArr = this.tagsArray;
      //adding the tag to the array
      //first if statement is checking that it's not a duplicate
      if (this.validateTag(this.currTag) === true) {
        //this if statement is checking if the tag you're adding is unresolved, if so it'll run this
        if (
          this.currTag === "unresolved" &&
          newArr.includes("resolved") === true
        ) {
          //grabbing resolved tag
          const rtag = "resolved";
          const indexOfRes = newArr.indexOf("resolved");
          //deletes resolved from the array
          newArr.splice(indexOfRes, 1);
          //adds unresolved tag
          newArr.push(this.currTag);
          //deletes resolved from the array
          axios
            .delete(`/api/tags/feedback/${this.id}/${rtag}`)
            .then(response => {
              this.removeTag(rtag);
              this.tagsArray = newArr;
            })
            .catch(error => {
              this.setError("There was a problem deleting the resolved tag. Please try again.");
            });
          //this if statement is checking if the tag you're adding is resolved, if so it'll run this
        } else if (
          this.currTag === "resolved" &&
          newArr.includes("unresolved") === true
        ) {
          //grabbing the unresolved tag
          const urtag = "unresolved";
          //finds where unresolved is in the array
          const indexOfUnres = newArr.indexOf("unresolved");
          //deletes unresolved from the array
          newArr.splice(indexOfUnres, 1);
          //adds resolved tag
          newArr.push(this.currTag);
          //deletes unresolved from the array
          axios
            .delete(`/api/tags/feedback/${this.id}/${urtag}`)
            .then(response => {
              this.removeTag(urtag);
            })
            .catch(error => {
              this.setError("There was a problem deleting the unresolved tag. Please try again.");
            });
          //if the tag isn't unresolved or resolved add it as usual
        } else if (this.currTag === "" || this.currTag === "") {
          const ptag = this.placeholderText.toLowerCase();
          if (ptag === "Add Tag") {
          } else {
            this.updated.push(ptag);
          }
        } else {
          newArr.push(this.currTag.toLowerCase());
        }
      }
      //grabbing the tag you just added
      const tag = newArr[newArr.length - 1];
      //validates the tag making sure it's not a dup
      this.validateTag(tag);
      this.currentItem = 0;
      // adding it to the array
      axios
        .put(`/api/tags/feedback/${this.id}/${tag}`)
        .then(response => {
          // Adding the tag to the card object
          this.addNewTag(tag);
          //mergeing the copy we the real array
          this.tagsArray = newArr;
        })
        .catch(error => {
          // console.log(error);
          this.setError("There was a problem adding the tag to the card object. Please try again.");
        });
      //after clicking enter it reset the field
      this.searchFilter = "";
      this.widthNum = 70;
    },
    //this adds the tag when you click an option in the dropdown menu
    //its long because of you have to consider the unresolved and resolved tag
    selectOption(option) {
      //option is an object
      this.selected = option;
      this.optionsShown = false;
      this.searchFilter = this.selected.name;
      this.$emit("selected", this.selected);
      const newArr = this.tagsArray;
      const tag = this.searchFilter;
      if (this.tagsArray.includes(tag) === false) {
        //this if statement is checking if the tag you're adding is unresolved, if so it'll run this
        if (tag === "unresolved" && newArr.includes("resolved") === true) {
          //grabbing resolved tag
          const rtag = "resolved";
          const indexOfRes = newArr.indexOf("resolved");
          //deletes resolved from the array
          newArr.splice(indexOfRes, 1);
          //adds unresolved tag
          newArr.push(tag);
          this.searchFilter = "";
          //deletes resolved from the array
          axios
            .delete(`/api/tags/feedback/${this.id}/${rtag}`)
            .then(response => {
              this.removeTag(rtag);
              this.tagsArray = newArr;
            })
            .catch(error => {
              this.setError("There was a problem deleting the resolved tag. Please try again.");
            });
          axios.put(`/api/tags/feedback/${this.id}/${tag}`)
            .then(response => {
              this.addNewTag(tag);
          })
            .catch(error => {
              this.setError("There was a problem adding the resolved tag. Please try again.");
            });
          //this if statement is checking if the tag you're adding is resolved, if so it'll run this
        } else if (
          tag === "resolved" &&
          newArr.includes("unresolved") === true
        ) {
          //grabbing the unresolved tag
          const urtag = "unresolved";
          //finds where unresolved is in the array
          const indexOfUnres = newArr.indexOf("unresolved");
          //deletes unresolved from the array
          newArr.splice(indexOfUnres, 1);
          //adds resolved tag
          newArr.push(tag);
          this.searchFilter = "";
          //deletes unresolved from the array
          axios
            .delete(`/api/tags/feedback/${this.id}/${urtag}`)
            .then(response => {
              this.removeTag(urtag);
            })
            .catch(error => {
              this.setError("There was a problem removing the unresolved tag. Please try again.");
            });
          axios.put(`/api/tags/feedback/${this.id}/${tag}`)
            .then(response => {
            this.addNewTag(tag);
          })
            .catch(error => {
              this.setError("There was a problem deleting unresolved tag. Pease try again.");
            });
          //this if statement is checking if the tag isn't unresolved or resolved add it as usual
        } else if (this.tagsArray.includes(tag) === false) {
          newArr.push(tag);
          this.searchFilter = "";
          axios
            .put(`/api/tags/feedback/${this.id}/${tag}`)
            .then(response => {
              // Adding the tag to the card object
              this.addNewTag(tag);
              this.tagsArray = newArr;
              this.selected = "";
            })
            .catch(error => {
              // console.log(error);
              this.setError("There was a problem adding a tag to the card object. Please try again.");
            });
        }
      }
      //after adding a tag keep the dropdowb menu open and reset the highlighted state
      this.optionsShown = true;
      this.currentItem = 0;
    },

    showOptions() {
      if (!this.disabled) {
        this.searchFilter = "";
        this.optionsShown = true;
      }
    },
    //if the drop down menu is open and want to close it click anywhere on page to close it
    //and reset fields
    exit() {
      if (!this.selected.id) {
        this.selected = {};
        this.searchFilter = "";
        this.currentItem = null;
        this.$refs.options.children[0].scrollIntoView({
        block: "nearest"
      })
      } else {
        this.searchFilter = this.selected.name;
      }
      this.searchFilter = "";
      this.$emit("selected", this.selected);
      this.optionsShown = false;
    },
    // Selecting when pressing Tab
    //auto fills the text box with whatever is highlighted
    keyMonitor: function(event) {
      this.searchFilter = this.filteredOptions[this.currentItem].name;
      this.currentItem = 0;
      this.optionsShown = true;
    },
    //scroll through the dropdown menu with arrow keys
    scrollArrows() {
      if (event.keyCode == 38 && this.currentItem > 0) {
        this.currentItem--;
        this.scrolling();
        // document.body.style.pointerEvents = this.pointerState
      } else if (
        event.keyCode == 40 &&
        this.currentItem < this.filteredOptions.length - 1
      ) {
        this.currentItem++;
        this.scrolling();
        // document.body.style.pointerEvents = this.pointerState
      }
    },
    //on mouseover/hover chaged what's selected and highlight to whatever option your mouse is on
    highlighted(index) {
      this.currentItem = index;
    },
    //enables you to see what is selected as you're scrolling with the arrow keys
    scrolling() {
      this.$refs.options.children[this.currentItem].scrollIntoView({
        block: "nearest"
      });
    }
  }
};
</script>
<style lang="scss" scoped>
@import "../../../../sass/sass_variables.scss";
.new-tag {
  padding: 0 0 0 0.6rem;
  height: 27px;
  font-size: 16px;
  padding-top: 5px;
  padding-bottom: 5px;
  color: $gray-700;
  outline: none;
  border: none;
  width: 100px;
  box-shadow: none;
  flex-grow: 1;
  // border-radius: 4px 0 0 4px;
}
.dropdown-content {
  position: absolute;
  background-color: $white;
  min-width: 248px;
  max-width: 248px;
  max-height: 200px;
  box-shadow: 0px -8px 34px 0px rgba(0, 0, 0, 0.05);
  overflow: auto;
  z-index: 1;
  border: 1px solid $gray-400;
  // border-bottom-left-radius: 4px;
  // border-bottom-right-radius: 4px;
}
.dropdown-item {
  color: $gray-700;
  line-height: 1em;
  text-decoration: none;
  display: block;
}
.dropdown:hover .dropdowncontent {
  display: block;
}
.highlightedOption {
  background-color: $primary-hover;
  color: $gray-800;
}
</style>
