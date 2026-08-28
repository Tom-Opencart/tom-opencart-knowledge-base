# Архитектура и Руководство по Swiper в теме Vita (OpenCart 3)

В теме **Vita** современный движок **Swiper** является единым стандартом для всех интерактивных фронтенд-компонентов: слайдеров баннеров, витрин товаров, отзывов, галерей, полноэкранных попапов (Lightbox) с зумом и мобильных навигационных лент.

---

## 📌 1. Карта сценариев использования Swiper в теме Vita

| Сценарий / Компонент | Шаблон / Скрипт | Ключевые возможности Swiper |
| :--- | :--- | :--- |
| **Главный баннер (Hero)** | `slideshow.twig` | **Parallax** (`data-swiper-parallax`), Autoplay с паузой при наведении (`pauseOnMouseEnter`), Dynamic bullets, Floating arrows |
| **Витрины товаров** | `vita_all_in_one.js`, `product.twig` | Пошаговое перелистывание (`freeMode: false`, `slidesPerGroup: 1`), кастомный скроллбар 170px (`moveDivider`), адаптивная сетка (1/2 товара на мобилках) |
| **Карточка товара (PDP)** | `product.twig`, `lightbox.js` | Мгновенная подмена фото по клику/ховеру миниатюр + плавный фейд |
| **Полноэкранный Lightbox (Zoom Popup)** | `lightbox.js` | **Pinch-to-Zoom**, Double-tap zoom, полноэкранная лента миниатюр, клавиатура (`ESC`, `←`, `→`), свайпы |
| **Галереи в статьях и модулях** | `vita_news_gallery.twig`, `gallery.twig` | Универсальная привязка `data-lightbox="group-name"`, открытие в полноэкранном Zoom-попапе |
| **Отзывы покупателей** | `vita_testimonial.js` | Адаптивная карусель с плавной прокруткой |
| **Мобильное меню / Чипсы категорий** | Перспективный паттерн | Горизонтальный скролл с инерцией (`freeMode: true`, `slidesPerView: 'auto'`) |
| **Истории акций (Stories)** | Перспективный паттерн | Полноэкранный сторис-попап с таймером прогресс-бара (`autoplay + progressbar`) |

---

## ⚙️ 2. Стандарты интеграции и Best Practices

### 2.1. Точечное подключение ассетов (Asset Performance)
Не подключать Swiper глобально в `header.twig`. Подключать только в контроллерах страниц и модулей, где он реально отображается:

```php
// В контроллере (product/product.php, vita_theme.php, vita_all_in_one.php и др.):
$this->document->addStyle('catalog/view/theme/vita/javascript/swiper/swiper-bundle.min.css');
$this->document->addScript('catalog/view/theme/vita/javascript/swiper/swiper-bundle.min.js');
```

В шаблонах использовать безопасную отложенную инициализацию с фоллбэком:
```javascript
if (typeof Swiper !== 'undefined') {
  initComponent();
} else {
  $.getScript('catalog/view/theme/vita/javascript/swiper/swiper-bundle.min.js', initComponent);
}
```

---

### 2.2. Защита от нулевых размеров в табах и попапах (Observer Guards)
Чтобы слайдеры внутри скрытых Bootstrap-вкладок (табы характеристик/отзывов) или всплывающих окон не ломались при открытии, **всегда** включать:

```javascript
observer: true,
observeParents: true,
observeSlideChildren: true
```

---

### 2.3. Изоляция инстансов без глобальных ID
В динамических модулях, где на одной странице может быть 2–4 карусели («Хиты», «Новинки», «Акции»):

```javascript
$('[data-vita-all-in-one-carousel]').each(function() {
  var $root = $(this);
  var $viewport = $root.find('.vita-all-in-one__viewport');
  
  $viewport.swiper({
    nextButton: $root.find('.vita-all-in-one__arrow--next'),
    prevButton: $root.find('.vita-all-in-one__arrow--prev'),
    scrollbar: $root.find('.vita-all-in-one__scrollbar')
  });
});
```

---

### 2.4. Плавное пошаговое перелистывание товаров в каруселях
Для предотвращения дерганий и произвольных прыжков ленты:
* **`freeMode: false`** — гарантирует, что стрелки навигации листают строго по 1 карточке (`slidesPerGroup: 1`, `speed: 350`).
* **`scrollbarSnapOnRelease: true`** — после перетаскивания мышью или скроллбаром карточка аккуратно «доводится» до границы сетки.

---

### 2.5. Фиксированная ширина ползунка скроллбара (170px)
Чтобы ползунок не растягивался в зависимости от количества товаров, Swiper задается фиксированный размер через JavaScript и CSS:

```javascript
function setScrollbarDragSize(swiper) {
  if (!swiper || !swiper.scrollbar) return;
  var scrollbar = swiper.scrollbar;
  var $root = $(swiper.container).closest('.vita-all-in-one');
  var $scrollbar = $root.find('.vita-all-in-one__scrollbar');
  var maximumTranslate = -swiper.maxTranslate();

  if (!maximumTranslate || maximumTranslate <= 0) {
    $scrollbar.hide();
    return;
  }
  $scrollbar.show();

  var trackSize = scrollbar.trackSize || $scrollbar.width();
  if (!trackSize) return;

  var fixedSize = 170;
  scrollbar.dragSize = fixedSize;
  scrollbar.moveDivider = (trackSize - fixedSize) / maximumTranslate;
  if (scrollbar.drag) {
    scrollbar.drag.css('width', fixedSize + 'px');
  }
  scrollbar.setTranslate();
}
```

```css
.swiper-scrollbar-drag {
  width: 170px !important;
  max-width: 170px !important;
  background: var(--mp-primary, #8C9D93) !important;
  border-radius: 10px;
}
```

---

### 2.6. Parallax в главном баннере (`slideshow.twig`)
Разделение фона и текста с разной скоростью смещения:

```twig
<div id="slideshow{{ module }}" class="swiper mp-slideshow">
  <div class="swiper-wrapper">
    {% for banner in banners %}
    <div class="swiper-slide mp-slideshow-slide">
      <a href="{{ banner.link }}" class="mp-slide-link">
        <div class="mp-slide-bg" data-swiper-parallax="-20%">
          <img src="{{ banner.image }}" alt="{{ banner.title }}" class="mp-slide-img" />
        </div>
        <div class="mp-slide-content" data-swiper-parallax="-300">
          <span class="mp-slide-title">{{ banner.title }}</span>
        </div>
      </a>
    </div>
    {% endfor %}
  </div>
</div>
```

---

### 2.7. Полноэкранный Поп-ап (Lightbox) с Zoom & Thumbs
Универсальный скрипт `lightbox.js` обслуживает любые фотогалереи на сайте:
* Принимает массив объектов `{ src, thumb, title, price }`.
* Содержит Zoom-контейнеры `<div class="swiper-zoom-container"><img src="..." /></div>`.
* Управляется стрелками навигации, клавиатурой, свайпами и нижней лентой миниатюр.
* Блокирует прокрутку фона страницы (`vitaLockPageScroll()`).
