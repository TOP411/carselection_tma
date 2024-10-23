<script setup>
import { ref, onMounted, computed } from 'vue'
import { XMLParser, XMLBuilder } from 'fast-xml-parser'

import { Swiper, SwiperSlide } from 'swiper/vue'
import { Navigation, Pagination } from 'swiper/modules'
import 'swiper/css'
import 'swiper/css/navigation'
import 'swiper/css/pagination'
const modules = [ Navigation, Pagination ]

const getFeed = async (url) => {
  try {
    const response = await fetch(`http://chatbotapi.top411.ru/feed/index.php?feed=${url}`)
    if (!response.ok) {
      throw new Error(`Response status: ${response.status}`);
    }
    const XMLdata = await response.text()
    const options = {
      ignoreAttributes: false,
      attributeNamePrefix : 'att_',
      textNodeName: 'text',
    }
    const parser = new XMLParser(options)
    let jObj = parser.parse(XMLdata)
    return jObj
  } catch (error) {
    console.log(error)
  }
}

const offers = ref([])

onMounted(async () => {
  const feed = await getFeed('https://media.cm.expert/stock/export/cmexpert/yml/all/all/f267397a2ae27ed4c75945deb8a11033.xml')
  if (feed) {
    offers.value = feed.yml_catalog?.shop?.offers?.offer
  }
})

const filters = ref(false)

const selectedParams = ref([]);

const filteredItems = computed(() => {
  return offers.value.filter(item => {
    return selectedParams.value.every(param => {
      return (
        !param.value || // Если значение не указано, пропускаем параметр
        item.param.some(itemParam =>
          itemParam.att_name === param.group && param.value === itemParam.text
        )
      );
    });
  });
});

const allGroupedParams = computed(() => {
  const grouped = {};
  const uniqueParams = {};

  offers.value.forEach(item => {
    item.param.forEach(param => {
      if (!uniqueParams[param.text]) {
        if (!grouped[param.att_name]) {
          grouped[param.att_name] = [];
        }
        grouped[param.att_name].push(param);
        uniqueParams[param.text] = true;
      }
    });
  });

  const result = Object.keys(grouped).map(att_name => ({
    group: att_name,
    value: grouped[att_name],
  }));
  
  // Отсортируем группы, чтобы модель всегда была первой
  return result.sort((a, b) => {
    if (a.group === 'Модель') return -1;
    if (b.group === 'Модель') return 1;
    return 0;
  });
});

// Обновленная функция toggleParam
const toggleParam = (att_name, text) => {
  // Если выбранная группа - 'Модель', сбрасываем другие параметры
  if (att_name === 'Модель') {
    selectedParams.value = []; // Сбрасываем все выбранные параметры
  }

  const index = selectedParams.value.findIndex(param => param.group === att_name && param.value === text);

  if (index > -1) {
    // Если параметр уже выбран, удаляем его
    selectedParams.value.splice(index, 1);
  } else {
    // Добавляем новый параметр
    selectedParams.value.push({ group: att_name, value: text });
  }
  
};

const isSelected = (group, text) => {
  return selectedParams.value.some(param => param.group === group && param.value === text);
};

const isDisabled = (group, text) => {
  return !filteredItems.value.some(item => 
    item.param.some(p => 
      p.att_name === group && p.text === text
    )
  ) && group !== 'Модель';
};

const resetParams = () => {
  selectedParams.value = []
}

const currentCar = ref({})
const currentCarDescription = ref(false)

// Метод для удаления параметров
const removeParam = (group, value) => {
  const index = selectedParams.value.findIndex(param => param.group === group && param.value === value);
  if (index > -1) {
    selectedParams.value.splice(index, 1);
  }
};

const formatSalesNote = (text) => {
  return text.replace(/(\d+)/g, (match) => {
    const number = parseInt(match);
    return number > 10000 ? new Intl.NumberFormat('ru-RU', { style: 'currency', currency: 'RUB', minimumFractionDigits: 0 }).format(number) : number
  })
}

const morph = (int, array) => {
  return (array = array || ['автомобиль', 'автомобиля', 'автомобилей']) && array[(int % 100 > 4 && int % 100 < 20) ? 2 : [2, 0, 1, 1, 1, 2][(int % 10 < 5) ? int % 10 : 5]]
}

</script>

<template>
  <div>

    <!-- хедер -->
    <header class="header">
      <button @click="filters = true">Фильтры</button>
      <div class="selected-params" v-if="selectedParams?.length > 0">
        <span v-for="(param, index) in selectedParams" :key="index" class="selected-param">
          {{ param.group }}: {{ param.value }}
          <button @click="removeParam(param.group, param.value)">&times;</button>
        </span>
      </div>
    </header>

    <!-- всплывающее окно с фильтрами -->
    <div
      v-if="filters"
      class="filter-container"
    >
      <button
        @click="filters = false"
        class="close-btn"
      >&times</button>
      <div v-for="(group, index) in allGroupedParams" :key="index" class="filter-group">
        <h3 class="filter-group-title">{{ group.group }}</h3>
        <div class="filter-buttons">
          <button
            v-for="(param, paramIndex) in group.value"
            :key="paramIndex"
            :class="{ selected: isSelected(group.group, param.text), model: group.group === 'Модель' }"
            :disabled="isDisabled(group.group, param.text)"
            @click="toggleParam(group.group, param.text)"
          >
            {{ param.text }}
          </button>
        </div>
      </div>

      <button class="reset-button" @click="resetParams()">Сброс</button>
      <div class="result-button">
        <button @click="filters = false" class="color-button">Показать {{ filteredItems.length }} {{ morph(filteredItems.length) }}</button>
      </div>

    </div>

    <!-- выдача результатов -->
    <div class="cars">
      <p class="result-count">Подходящих автомобилей: {{ filteredItems.length }}</p>
      <div class="car-list">
        <div v-for="item in filteredItems" :key="item.att_id" class="car-wrapper">
          <img class="car-image" :src="Array.isArray(item.picture) ? item.picture[0] : item.picture" :alt="item.name">
          <p class="car-name">{{ item.name }}</p>
          <!-- <div
            v-html="item.sales_notes?.replace(/\n/g, '<br>')"
            class="sales-notes"
          ></div> -->
          <button class="color-button" @click="currentCar = item">Подробнее</button>
        </div>
      </div>
    </div>

    <!-- всплывающее окно моделей -->
    <div
      v-if="currentCar.model"
      class="current-car"
    >
      <button
        @click="currentCar = {}"
        class="close-btn"
      >&times;</button>
      <swiper
        v-if="Array.isArray(currentCar.picture)"
        :modules="modules"
        :slides-per-view="1"
        :space-between="50"
        :navigation="{enabled: true}"
        :pagination="{ clickable: true }"
        :autoHeight="true"
      >
        <swiper-slide
          v-for="picture in currentCar.picture"
        >
          <img :src="picture" :alt="currentCar.name">
        </swiper-slide>
      </swiper>
      <img v-else :src="currentCar.picture" :alt="currentCar.name">
      <h1>{{ currentCar.name }}</h1>
      <table
        class="param-table"
        cellpadding="4"
      >
        <tr
          v-for="param, i in currentCar.param"
          :key="`param-${i}`"
        >
          <td align="right">{{ param.att_name }}</td>
          <td>{{ param.text }}</td>
        </tr>
      </table>
      <div>
        <div v-html="currentCarDescription ? currentCar.description?.replace(/\n/g, '<br>') : currentCar.description?.replace(/\n/g, '<br>')?.slice(0, 348) + ' ...'"></div>
        <button
          class="description-button"
          @click="currentCarDescription = !currentCarDescription"
        >
          <u>{{ currentCarDescription ? 'Скрыть описание' : 'Развернуть описание' }}</u>
        </button>
      </div>
      

      <div
        v-html="formatSalesNote(currentCar.sales_notes)?.replace(/\n/g, '<br>')"
        class="sales-notes"
      ></div>
      
      <div class="submit-form">
        <input type="text" placeholder="Номер телефона">
        <button class="color-button">Рассчитать стоимость</button>
      </div>
    </div>

  </div>
</template>


<style>


</style>
