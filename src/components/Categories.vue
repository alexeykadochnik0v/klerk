<template>
  <div class="p-6 bg-gray-100 min-h-screen">
    <!-- Загрузка индикатора -->
    <div v-if="loading" class="flex justify-center items-center min-h-screen">
      <div class="animate-spin rounded-full h-16 w-16 border-t-4 border-blue-600"></div>
    </div>

    <!-- Основное содержимое -->
    <div v-else>
      <!-- Чекбокс для отображения пустых рубрик -->
      <div class="mb-6 flex items-center space-x-2">
        <input type="checkbox" id="allow-empty" v-model="showEmpty" @change="fetchRubrics"
          class="w-4 h-4 text-blue-600 rounded focus:ring-blue-500 border-gray-300" />
        <label for="allow-empty" class="text-gray-700 font-medium">Отображать пустые рубрики</label>
      </div>

      <!-- Отображение суммы мероприятий по отмеченным чекбоксам -->
      <div class="bg-white shadow-md rounded-md p-4 mb-6">
        <h2 class="text-lg font-bold text-gray-800">Сумма выбранных мероприятий: <span class="text-blue-600">{{
            selectedCountSum }}</span></h2>
      </div>

      <!-- Дерево рубрик -->
      <div class="bg-white shadow-md rounded-md p-4">
        <ul>
          <li v-for="rubric in rubrics" :key="rubric.id" class="mb-4">
            <div class="flex items-center space-x-3">
              <!-- Кнопка сворачивания/разворачивания с анимацией -->
              <button @click="toggle(rubric)" class="focus:outline-none transition-transform transform duration-300">
                <span v-if="rubric.isOpen" class="text-gray-500 rotate-180">&#9660;</span>
                <span v-else class="text-gray-500">&#9658;</span>
              </button>

              <!-- Чекбокс для подсчета мероприятий -->
              <input type="checkbox" :id="rubric.id" v-model="rubric.isSelected"
                class="w-4 h-4 text-blue-600 rounded focus:ring-blue-500 border-gray-300"
                @change="updateSelectedState(rubric)" />

              <!-- Ссылка на рубрику -->
              <a :href="`https://www.klerk.ru${rubric.url}`" class="text-blue-600 font-medium hover:underline"
                target="_blank">
                {{ rubric.title }}
              </a>
              <span class="text-gray-500">({{ rubric.count }} / {{ getTotalCount(rubric) }})</span>
            </div>

            <!-- Дочерние рубрики с анимацией -->
            <transition name="slide-fade" mode="out-in">
              <ul v-if="rubric.isOpen && rubric.children && rubric.children.length > 0" class="ml-6 mt-2">
                <li v-for="child in rubric.children" :key="child.id" class="mb-2">
                  <div class="flex items-center space-x-3">
                    <button @click="toggle(child)"
                      class="focus:outline-none transition-transform transform duration-300">
                      <span v-if="child.isOpen" class="text-gray-500 rotate-180">&#9660;</span>
                      <span v-else class="text-gray-500">&#9658;</span>
                    </button>

                    <input type="checkbox" :id="child.id" v-model="child.isSelected"
                      class="w-4 h-4 text-blue-600 rounded focus:ring-blue-500 border-gray-300"
                      @change="updateSelectedState(child)" />

                    <a :href="`https://www.klerk.ru${child.url}`" class="text-blue-600 font-medium hover:underline"
                      target="_blank">
                      {{ child.title }}
                    </a>
                    <span class="text-gray-500">({{ child.count }} / {{ getTotalCount(child) }})</span>
                  </div>

                  <!-- Вложенные дочерние рубрики с анимацией -->
                  <transition name="slide-fade" mode="out-in">
                    <ul v-if="child.isOpen && child.children && child.children.length > 0" class="ml-6 mt-2">
                      <li v-for="grandChild in child.children" :key="grandChild.id" class="mb-2">
                        <div class="flex items-center space-x-3">
                          <input type="checkbox" :id="grandChild.id" v-model="grandChild.isSelected"
                            class="w-4 h-4 text-blue-600 rounded focus:ring-blue-500 border-gray-300"
                            @change="updateSelectedState(grandChild)" />

                          <a :href="`https://www.klerk.ru${grandChild.url}`"
                            class="text-blue-600 font-medium hover:underline" target="_blank">
                            {{ grandChild.title }}
                          </a>
                          <span class="text-gray-500">({{ grandChild.count }} / {{ getTotalCount(grandChild) }})</span>
                        </div>
                      </li>
                    </ul>
                  </transition>
                </li>
              </ul>
            </transition>
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';

export default {
  data() {
    return {
      rubrics: [],
      showEmpty: false,
      loading: true,
    };
  },
  computed: {
    selectedCountSum() {
      let sum = 0;
      const calculateSelectedSum = (rubric) => {
        if (rubric.isSelected) {
          sum += rubric.count;
        }
        if (rubric.children && rubric.children.length > 0) {
          rubric.children.forEach((child) => calculateSelectedSum(child));
        }
      };
      this.rubrics.forEach((rubric) => calculateSelectedSum(rubric));
      return sum;
    },
  },
  methods: {
    fetchRubrics() {
      this.loading = true;
      const allowEmptyParam = this.showEmpty ? '?allowEmpty=1' : '';
      axios
        .get(`https://www.klerk.ru/yindex.php/v3/event/rubrics${allowEmptyParam}`)
        .then((response) => {
          this.rubrics = this.processRubrics(response.data);
        })
        .catch((error) => {
          console.error('Ошибка при загрузке рубрик:', error);
        })
        .finally(() => {
          this.loading = false;
        });
    },
    processRubrics(rubrics) {
      return rubrics.map((rubric) => ({
        ...rubric,
        isOpen: false,
        isSelected: false,
        children: rubric.children ? this.processRubrics(rubric.children) : [],
      }));
    },
    toggle(rubric) {
      rubric.isOpen = !rubric.isOpen;
    },
    updateSelectedState(rubric) {
      // Рекурсивно изменяем состояние всех дочерних элементов при изменении родительского чекбокса
      const updateChildren = (rubric, isSelected) => {
        rubric.isSelected = isSelected;
        if (rubric.children && rubric.children.length > 0) {
          rubric.children.forEach((child) => updateChildren(child, isSelected));
        }
      };
      updateChildren(rubric, rubric.isSelected);
    },
    getTotalCount(rubric) {
      let totalCount = rubric.count || 0;
      if (rubric.children && rubric.children.length > 0) {
        rubric.children.forEach((child) => {
          totalCount += this.getTotalCount(child);
        });
      }
      return totalCount;
    },
  },
  mounted() {
    this.fetchRubrics();
  },
};
</script>

<style scoped>
/* Дополнительные стили для улучшения интерфейса */
.button {
  cursor: pointer;
  transition: color 0.2s;
}

.button:hover {
  color: #1d4ed8;
}

/* Анимация для сворачивания/разворачивания */
.slide-fade-enter-active,
.slide-fade-leave-active {
  transition: all 0.3s ease;
}

.slide-fade-enter-from,
.slide-fade-leave-to {
  transform: translateY(-10px);
  opacity: 0;
}
</style>