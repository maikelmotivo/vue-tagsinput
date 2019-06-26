<template>
    <div class="tags-input-root">
        <div :class="wrapperClass + ' tags-input'" @click.self="focusInput()">
            <span class="tags-input-badge tags-input-badge-pill tags-input-badge-selected-default"
                v-for="(badge, index) in tagBadges"
                :key="index"
            >
                <span v-html="badge"></span>

                <i href="#" class="tags-input-remove" @click.prevent="removeTag(index)" v-if="!locked"></i>
            </span>

            <input type="text"
                ref="taginput"
                :placeholder="getPlaceholder()"
                v-model="input"
                :readonly="locked"
                @keydown.enter.prevent="tagFromInput"
                @keydown.8="removeLastTag"
                @keydown.down="nextSearchResult"
                @keydown.up="prevSearchResult"
                @keydown="onKeyDown"
                @keyup="searchTag"
                @keyup.esc="ignoreSearchResults"
                @focus="onFocus"
                @blur="onBlur"
                @value="tags">

            <input type="hidden" v-if="elementId"
                :name="elementId"
                :id="elementId"
                v-model="hiddenInput">
        </div>

        <!-- Typeahead/Autocomplete -->
        <div v-show="searchResults.length">
            <p v-if="typeaheadStyle === 'badges'" :class="`typeahead-${typeaheadStyle}`">
                <span v-for="(tag, index) in searchResults"
                    :key="index"
                    v-html="tag.text + tag.lock"
                    @mouseover="searchSelection = index"
                    @mousedown.prevent="tagFromSearchOnClick(tag)"
                    class="tags-input-badge"
                    v-bind:class="{
                        'tags-input-typeahead-item-default': index != searchSelection,
                        'tags-input-typeahead-item-highlighted-default': index == searchSelection
                    }"></span>
            </p>

            <ul v-else-if="typeaheadStyle === 'dropdown'"
              :class="`typeahead-${typeaheadStyle}`"
              v-bind:style="typeaheadStyleObject"
            >
                <li v-for="(tag, index) in searchResults"
                :key="index"
                v-html="tag.text"
                @mouseover="searchSelection = index"
                @mousedown.prevent="tagFromSearchOnClick(tag)"
                v-bind:class="{
                    'tags-input-typeahead-item-default': index != searchSelection,
                    'tags-input-typeahead-item-highlighted-default': index == searchSelection
                }"></li>
            </ul>
        </div>
    </div>
</template>

<script>
export default {
    props: {
        elementId: String,

        existingTags: {
            type: Object,
            default: () => {
                return {};
            }
        },

        value: {
            type: [Array, String],
            default: () => {
                return [];
            }
        },

        typeahead: {
            type: Boolean,
            default: false
        },

        typeaheadStyle: {
            type: String,
            default: 'badges'
        },

        typeaheadActivationThreshold: {
            type: Number,
            default: 1
        },

        typeaheadMaxResults: {
            type: Number,
            default: 0
        },

        placeholder: {
            type: String,
            default: 'Add a tag'
        },

        lockedPlaceholder: {
            type: String,
            default: 'Tags are locked'
        },

        limit: {
            type: Number,
            default: 0
        },

        onlyExistingTags: {
            type: Boolean,
            default: false
        },

        deleteOnBackspace: {
            type: Boolean,
            default: true
        },

        allowDuplicates: {
            type: Boolean,
            default: false
        },

        validate: {
            type: Function,
            default: () => true
        },

        addTagsOnComma: {
            type: Boolean,
            default: false
        },

        wrapperClass: {
            type: String,
            default: 'tags-input-wrapper-default'
        },

        typeaheadStyleObject: {
            type: Object,
            default: {}
        },

        sortSearchResults: {
            type: Boolean,
            default: true
        },

        locked: {
            type: Boolean,
            default: false,
        },

        followingTags: {
            type: Array,
            default: function() { return [] },
        },
    },

    data() {
        return {
            badgeId: 0,
            tagBadges: [],
            tags: [],

            input: '',
            oldInput: '',
            hiddenInput: '',

            searchResults: [],
            searchSelection: 0,
        };
    },

    created () {
        this.tagsFromValue();

        // Emit an event
        this.$emit('initialized');
        
        this.$root.$on('change-input', (input) => {
            this.input = input;
        });

        this.$root.$on('tag-from-input', (searchResults) => {
            if (!searchResults) {
                this.searchResults = [];
            }
            this.tagFromInput();
        });
    },

    watch: {
        input() {
            this.$emit('input-changed', this.input);
        },

        tags() {
            // Updating the hidden input
            this.hiddenInput = this.tags.join(',');

            // Update the bound v-model value
            this.$emit('input', this.tags);
        },

        value() {
            this.tagsFromValue();
        }
    },

    methods: {
        getPlaceholder() {
            return this.locked ? this.lockedPlaceholder : this.placeholder;
        },
        escapeRegExp(string) {
            return string.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
        },

        tagFromInput() {
            // If we're choosing a tag from the search results
            if (this.searchResults.length && this.searchSelection >= 0) {
                this.tagFromSearch(this.searchResults[this.searchSelection]);

                this.input = '';
            } else {
                // If we're adding an unexisting tag
                let text = this.input.trim();

                // If the new tag is not an empty string and passes validation
                if (!this.onlyExistingTags && text.length && this.validate(text)) {
                    this.input = '';

                    // Determine the tag's slug and text depending on if the tag exists
                    // on the site already or not
                    let slug = this.makeSlug(text);
                    let existingTag = this.existingTags[slug];

                    slug = existingTag ? slug : text;
                    text = existingTag ? existingTag : text;
                    let tag = existingTag ? existingTag : {
                        name: slug,
                        private: 0,
                    }

                    this.addTag(slug, text);
                }
            }
        },

        tagFromSearchOnClick(tag) {
            this.tagFromSearch(tag);
        },

        tagFromSearch(tag) {
            this.searchResults = [];
            this.input = '';
            this.oldInput = '';

            let t = {
                name: tag.slug,
                private: 0,
            }

            this.addTag(tag.slug, tag.text);
        },

        makeSlug(value) {
            return value.toLowerCase().replace(/\s/g, '-');
        },

        hasPrivateTag() {
            let self = this;
            return this.tags.map(function (slug) {
                if (typeof self.existingTags[slug] !== 'undefined' && self.existingTags[slug].private === 1) {
                    return 1;
                }

                return 0;
            }).includes(1);
        },

        addTag(slug, text) {
            // Check if the limit has been reached
            if (this.limit > 0 && this.tags.length >= this.limit) {
                return false;
            }

            let isPrivate = typeof this.existingTags[slug] !== 'undefined' && this.existingTags[slug].private === 1;

            if (isPrivate && ! this.existingTags[slug].following) {
                this.$emit('tag-inaccessible', slug);

                return;
            }

            if ((isPrivate && this.tags.length > 0) || this.hasPrivateTag()) {
                this.$emit('max-one-private-tag', slug);

                return;
            }

            // Attach the tag if it hasn't been attached yet
            if (!this.tagSelected(slug)) {
                this.tagBadges.push(text);
                this.tags.push(slug);
            }

            // Emit events
            this.$emit('tag-added', slug);
            this.$emit('tags-updated');
        },

        removeLastTag() {
            if (!this.input.length && this.deleteOnBackspace) {
                this.removeTag(this.tags.length - 1);
            }
        },

        removeTag(index) {
            let slug = this.tags[index];

            this.tags.splice(index, 1);
            this.tagBadges.splice(index, 1);

            // Emit events
            this.$emit('tag-removed', slug);
            this.$emit('tags-updated');
        },

        searchTag() {
            if (this.typeahead === true) {
                if (this.oldInput != this.input || (!this.searchResults.length && this.typeaheadActivationThreshold == 0)) {
                    this.searchResults = [];
                    this.searchSelection = 0;
                    let input = this.input.trim();

                    if ((input.length && input.length >= this.typeaheadActivationThreshold) || this.typeaheadActivationThreshold == 0) {
                        for (let slug in this.existingTags) {
                            let text = this.existingTags[slug].text.toLowerCase();

                            if (text.search(this.escapeRegExp(input.toLowerCase())) > -1 && ! this.tagSelected(slug)) {
                                this.searchResults.push({ slug, text: this.existingTags[slug].text + this.existingTags[slug].lock, private: this.existingTags[slug].private });
                            }
                        }

                        // Sort the search results alphabetically
                        if (this.sortSearchResults) {
                          this.searchResults.sort((a, b) => {
                              if (a.text < b.text) return -1;
                              if (a.text > b.text) return 1;

                              return 0;
                          });
                        }

                        // Shorten Search results to desired length
                        if (this.typeaheadMaxResults > 0) {
                            this.searchResults = this.searchResults.slice(
                                0,
                                this.typeaheadMaxResults
                            );
                        }
                    }

                    this.oldInput = this.input;
                }
            }
        },
        
        onBlur() {
          this.$parent.$emit('tags-input-blur');
          this.hideTypeahead();
        },

        onFocus() {
            this.searchTag();
        },

        hideTypeahead() {
            if (! this.input.length) {
                this.$nextTick(() => {
                    this.ignoreSearchResults();
                });
            }
        },

        nextSearchResult() {
            if (this.searchSelection + 1 <= this.searchResults.length - 1) {
                this.searchSelection++;
            }
        },

        prevSearchResult() {
            if (this.searchSelection > 0) {
                this.searchSelection--;
            }
        },

        ignoreSearchResults() {
            this.searchResults = [];
            this.searchSelection = 0;
        },

        /**
        * Clear the list of selected tags
        */
        clearTags() {
            this.tags.splice(0, this.tags.length);
            this.tagBadges.splice(0, this.tagBadges.length);
        },

        /**
        * Replace the currently selected tags with the tags from the value
        */
        tagsFromValue() {
            if (this.value && this.value.length) {
                let tags = Array.isArray(this.value)
                    ? this.value
                    : this.value.split(',');

                if (this.tags == tags) {
                    return;
                }

                this.clearTags();

                for (let slug of tags) {
                    let existingTag = this.existingTags[slug];
                    let text = existingTag ? existingTag.text : slug;

                    this.addTag(slug, text);
                }
            } else {
                if (this.tags.length == 0) {
                    return;
                }

                this.clearTags();
            }
        },

        focusInput() {
            this.$refs.taginput.focus();
        },

        /**
        * Check if the tag with the provided slug is already selected
        */
        tagSelected(slug) {
            if (this.allowDuplicates) {
                return false;
            }

            if (! slug) {
                return false;
            }

            let searchSlug = this.makeSlug(slug);
            let found = this.tags.find((value) => {
                return searchSlug == this.makeSlug(value);
            });

            return !! found;
        },

        /**
         * Process all the keydown events
         */
        onKeyDown(e) {
            // Insert a new tag on comma keydown if the option is enabled
            if (e.key == ',') {
                if (this.addTagsOnComma) {
                    // The comma shouldn't actually be inserted
                    e.preventDefault();

                    // Add the inputed tag
                    this.tagFromInput();
                }
            }
        },
    }
}
</script>

<style>
.tags-input-root {
    position: relative;
}
</style>
