<!--
  - Copyright (c) 2020 Martin Denham, Tuomas Airaksinen and the And Bible contributors.
  -
  - This file is part of And Bible (http://github.com/AndBible/and-bible).
  -
  - And Bible is free software: you can redistribute it and/or modify it under the
  - terms of the GNU General Public License as published by the Free Software Foundation,
  - either version 3 of the License, or (at your option) any later version.
  -
  - And Bible is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY;
  - without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
  - See the GNU General Public License for more details.
  -
  - You should have received a copy of the GNU General Public License along with And Bible.
  - If not, see http://www.gnu.org/licenses/.
  -->

<template>
  <div :style="`height:${config.toolbarOffset}px`"/>
  <div id="notes"/>
  <div v-if="config.developmentMode"
      :style="`position: fixed; top:0; width:100%;  background-color: rgba(100, 255, 100, 0.7);
               height:${config.toolbarOffset}px`"
  >
     Current verse: {{currentVerse}}
  </div>
  <div v-if="config.developmentMode" class="highlightButton"><span @click="highLight">Highlight!</span> <span @mouseenter="getSelection">Get selection!</span></div>
  <div id="top" ref="topElement" :style="styleConfig">
    <div v-for="({key, contents}, index) in osisFragments" :key="key || index">
      <template v-for="(content, idx) in contents" :key="`${key}-${idx}`">
        <OsisFragment :content="content"/>
        <div v-if="contents.length > 0 && idx < contents.length" class="divider" />
      </template>
    </div>
  </div>
  <div id="bottom"/>
</template>
<script>
  import OsisFragment from "@/components/OsisFragment";
  import {provide, reactive, watch} from "@vue/runtime-core";
  import highlightRange from "dom-highlight-range";
  import {useAndroid, useBookmarks, useConfig, useStrings, useVerseNotifier} from "@/composables";
  import {testData} from "@/testdata";
  import {ref} from "@vue/reactivity";
  import {useInfiniteScroll} from "@/code/infinite-scroll";

  function findElemWithOsisID(elem) {
    if(elem === null) return;
    // This needs to be done unique for each OsisFragment (as there can be many).
    if(elem.dataset && elem.dataset.osisID) {
      return elem;
    }
    else {
      return findElemWithOsisID(elem.parentElement);
    }
  }

  export default {
    name: "BibleView",
    components: {OsisFragment},
    setup() {
      const {config} = useConfig();
      const strings = useStrings();
      const android = useAndroid();
      const osisFragments = reactive([]);
      const topElement = ref(null);
      useInfiniteScroll(config, android, osisFragments);
      const {currentVerse} = useVerseNotifier(config, android, topElement);
      const bookmarks = useBookmarks();

      watch(() => osisFragments, () => {
        for(const frag of osisFragments) {
            bookmarks.updateBookmarks(frag.bookmarks);
            bookmarks.updateBookmarkLabels(frag.bookmarkLabels);
        }
      }, {deep: true});

      window.bibleViewDebug.osisFragments = osisFragments;

      function replaceOsis(...s) {
        osisFragments.splice(0)
        osisFragments.push(...s)
      }

      if(process.env.NODE_ENV === "development") {
        console.log("populating test data");
        replaceOsis(...testData)
      }

      window.bibleView.replaceOsis = replaceOsis;
      window.bibleView.setTitle = (title) => {
        document.title = `${title} (${process.env.NODE_ENV})`;
      }

      provide("bookmarks", bookmarks);
      provide("config", config);
      provide("strings", strings);
      provide("android", android);
      console.log("android", android);
      return {config, strings, osisFragments, topElement, currentVerse};
    },
    computed: {
      styleConfig({config}) {
        let style = `
          max-width: ${config.marginSize.maxWidth};
          color: ${config.textColor};
          hyphens: ${config.hyphenation ? "auto": "none"};
          noise-opacity: ${config.noiseOpacity/100};
          line-spacing: ${config.lineSpacing / 10}em;
          line-height: ${config.lineSpacing / 10}em;
          text-align: ${config.justifyText ? "justify" : "start"};
          `;
        if(config.marginSize.marginLeft || config.marginSize.marginRight) {
          style += `
            margin-left: ${config.marginSize.marginLeft}mm;
            margin-right: ${config.marginSize.marginRight}mm;
          `;
        }
        return style;
      }
    },
    methods: {
      highLight() {
        //const first = document.getElementById("2Thess.2.12");
        //const second = document.getElementById("2Thess.2.15");
        const startCount = 16;
        const endCount = 53;
        const startOff = 75;
        const endOff = 78;
        const first = document.querySelector(`[data-element-count="${startCount}"]`).childNodes[0];
        const second = document.querySelector(`[data-element-count="${endCount}"]`).childNodes[0];
        const range = new Range();
        range.setStart(first, startOff);
        range.setEnd(second, endOff);
        const removeHighlights = highlightRange(range, 'span', { class: 'highlighted' });
      },
      getSelection() {
        const selection = window.getSelection();
        if (!selection.isCollapsed) {
          const range = selection.getRangeAt(0);
          console.log("range", range);

          const startElem = findElemWithOsisID(range.startContainer);
          const endElem = findElemWithOsisID(range.endContainer);

          console.log(
              startElem.dataset.osisID,
              startElem.dataset.elementCount,
              range.startOffset
          );
          console.log(
              endElem.dataset.osisID,
              endElem.dataset.elementCount,
              range.endOffset
          );
        }
      }
    },
  }
</script>
<style>
.highlighted {
  background-color: yellow;
}
.highlightButton {
  position: fixed;
  bottom: 0;
  left: 0;
  background: yellow;
}
.inlineDiv {
  display: inline;
}

.divider {
  height: 1em;
}

#bottom {
  padding-bottom: 100vh;
}
</style>