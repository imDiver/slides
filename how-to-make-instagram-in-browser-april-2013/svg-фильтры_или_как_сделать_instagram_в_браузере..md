Как сделать Instagram в браузере. 

Однажды у нас возникла задача сделать в веб-приложении филтры для изображений на стороне клиента. Под фиьтрами, мы будем подразумевать различные графические эффекты, подавление шума, смешивание слоев, линейные и нелинейные искажения картинки.

2-3 года назад можно было с уверенностью сказать, что существует лишь 2 способа добиться подобных эффектов.

1. Flash(Вечная ему память, ибо нам нужно поддерживать мобильные браузеры)
2. Отправлять изображение на сервер, там фильтровать, возвращать обратно на клиент и все это время ждать, ждать, ждать...

Но время идет, веб-стандарты развиваются и мы решили посмотреть, какие средства есть в арсенале фронтенд-разработчика сейчас.
После небольшого обзора и некоторых эксперементов, на данный момент (лето 2014), я насчитал 6 возможных способов фильтрации изображений в браузере:

1. Canvas
2. WebGL
3. CSS-фильтры
4. CSS-шейдеры
5. SVG-фильтры
6. SVG-фильтры для HTML 

Давайте рассмотрим их все по порядку, а так-же выделим достоинства и недостатки каждого похода.


## Canvas

Здесь все просто, логично и по своему прекрасно. Для того что-бы применить эффекты к изображению Вам достаточно положить его в canvas и математически описать преобразования, которые нужно сделать с каждым пикселем.

#### js

    // создаем или находим canvas
    var canvas = document.createElement('canvas'); 
    // получаем его 2D контекст
    var context = canvas.getContext('2d'); 
    // помещаем изображение в контекст 
    context.drawImage(img); 
    // получаем объект, описывающий внутреннее состояние области контекста
    var imgd = context.getImageData(0, 0, w, h); 
    // получаем одномерный массив, описывающий все пиксели изображения
    var pixels = imgd.data;

Мы получили `pixels`, одномерный массив в котором последовательно представлены значения красного, зеленого, синего канала, а так-же канала прозрачности для каждого пикселя изображения. Графически его можно представить так: 

     
<img>

TODO: - как обратно

### Достоинства фильтрации с Canvas

- поддерживается IE 9-ым и пракически всеми мобильными браузерами.
- мнодество готовых решений, библиотек, плагинов.
- фильтры могут быть настолько сложными и нестандартными, насколько у вас хватит фантазии и насколько вы разбираетесь в цифровой обработке изображений :)

### Недостатки фильтрации с Canvas

 - Нельзя отфильтровать картинки с других доменов (включая поддомены) из-за ограничений безопасности браузера. Это довольно легко решается проксированием или переводом в base64, но как бы то ни было, создает дополнительные проблемы. 
 - Если мы говорим про сложные фильтры, то это медленная, последовательная, блокирующая операция. Десктопный браузер при этом подтормаживает… а мобильный браузер серьезно тупит.
 


## 3. WebGl: Фильтрация 2D изображений изображений с огромной скоростью. Из серии "Как сделать Инстаграм в браузере".

Эта 3-я часть статьи из серии "Как сделать Инстаграм в браузере".

Другие статьи из серии: 

1. **Основные понятия**: Обзор возможных способов фильтрации изображений в веб.
2. **Canvas**: Как применить эффекты к изображению при помощи canvas?
3. **WebGl**: Фильтрация 2D изображений изображений с огромной скоростью.
4. **CSS-фильтры**: Быстрые css эффекты красивым синтаксисом.
5. **CSS-шейдеры**: Гримучая смесь GLSL и CSS. Будущее или тупик?
6. **SVG-фильтры**: Красивый неизведанный мир с его устоями, синтаксисом и правилами.
7. **SVG-фильтры для HTML**: Эффекты и фильтры для веб-контента с svg-синтаксисом.




Если сказать коротко, WebGl это браузерный API постренный на принципах OpenGl и позволяющий использовать все прелести графического процессора (GPU) в браузере. Применимо к нашей задаче, основное  достоинство WebGl - параллельная обработка всех пискелей изобржения. В отличиие от Canvas, нам не нужно последовательно в цикле проходить по всему изображению и производить вычисления для каждого пикселя. 

Дело в том, что у GPU (графического процессора), в отличие от CPU (центрального процессора) есть десятки тысяч ядер, которые могут работать параллельно. Вот хорошее старое видео наглядно поясняющее разницу в производительности между GPU и CPU.




Таким образом спользуя WebGL для фильтрации изображений, мы отдаем каждый пиксель изображения отдельному ядру GPU, которое обрабатывает его по определенным нами законам. А законы эти описываются вот такими, на первый взгляд страшными кусками кода:

#### wtf?

    precision mediump float;
    varying vec2 position;
    uniform sampler2D webcam;

    void main() {
      vec2 pos = position;
      vec4 color = texture2D(webcam, pos);
      color.rgb = 1.0 - color.rgb;
      gl_FragColor = color;
    }

Кто-нибудь знает что это такое?    
Правильно! Это шейдер - кусок кода в синтаксисе GLSL (OpenGL Shading Language), который описывает алгоритм обработки каждого пикселя в GPU. 

Шейдеры бывают

1. Векторными (Vertex Shaders) для работы с вершинами и полигонами в 3D.
2. Пиксельными (Fragment Shaders) для работы с текстурами.

В рамках задачи фильтрации изображений нас будут интересовать только пиксельные шейдеры.

### Не нужно бояться.

Давайте избавимся от страха перед синтаксисом и уточним, что вышеописанный шейдер всего лишь инвертирует изображение. Вся его суть заключена в этой строчке:

    color.rgb = 1.0 - color.rgb;
    
Весь осталной код - объявление переменных, описание источника изображения и значений на выходе. 

**Вот несколько особенностей языка GLSL:**

 - Все значения находятся в диапазоне 0..1 
 - GLSL позволяет получить доступ к компонентам векторов с помощью букв `x`, `y`, `z`, `w` и `r`, `g`, `b`, `a`. Таким образом, для двумерного изображения (vec2) можно использовать `pos.x`, `pos.y`. Для цвета (vec4) можно использовать `color.r`, `color.g`, `color.b`, `color.a`
 - GLSL поддерживает сокращенные записи. Например `color.rgb` (работа сразу с красным, зеленым и синим каналом), `color.rа` (только красный канал и канал прозрачности) и т.д. Это просо приятный синтаксический сахар. Запись `color.rgb  = 1 - color.rgb;` аналогична записи `color.r  = 1 - color.r; color.g  = 1 - color.g; color.b  = 1 - color.b;`. 
 - Большинство функций GLSL может работать с несколькими типами входных параметров (float, vec2, vec3 и vec4).
 - Отладка GLSL нелегкая задача, однако JS консоль Chrom-a довольно хорошо сообщает об ошибках и указывает на строку шейдера, которая вызывает проблему.

### A и куда этот шейдер писать?

Вот здесь я могу точно сказать - не изобретайте собственный велосипед. WebGl имеет довольно высокий порог вхождения. Он призван решать задачи на порядок сложнее фильтрации 2D изображений. Вы потратите много драгоценного времени на настройку и установку сцены, прежде чем сумеете применить тот самый шейдер. Это чем-то похоже на убиванием мухи из пулемета.

Гораздо проще и приятнее использовать готовые библиотеки, для фильтрации изображеий на основе WebGL. Давайте рассмотрим некоторые из них

### GLFX

GLFX - готовая библиотека, работающая с шейдерами для фильтрации 2D изображений

Демо: [evanw.github.io/glfx.js/demo/](http://evanw.github.io/glfx.js/docs/)
Исходники на гитхабе: [github.com/evanw/glfx.js](https://github.com/evanw/glfx.js)
Документация: [evanw.github.io/glfx.js/docs/](http://evanw.github.io/glfx.js/docs/)

	var canvas = fx.canvas();
	// convert the image to a texture
	var image = document.getElementById('image');
	var texture = canvas.texture(image);
	
	// apply the ink filter
	canvas.draw(texture)
	.sepia(0.34)
	.brightnessContrast(0.5, -0.5)
	.ink(0.25)
	.update();

**Достоинства библиотеки glfx:**

 - Красивый jQuery-подобный API с цепочками преобразований.
 - Возможность расширять библиотеку своими шейдерами.

### WebGLImageFilter


WebGLImageFilter - красивая библиотека для фильтрации 2D изображений на базе WebGl с множеством предустановленных шейдеров. 

Демо: [phoboslab.org/log/2013/11/fast-image-filters-with-webgl](http://phoboslab.org/log/2013/11/fast-image-filters-with-webgl)
Исходники на гитхабе: [github.com/phoboslab/WebGLImageFilter](https://github.com/phoboslab/WebGLImageFilter)


WebGLImageFilter прост в использовании:

    var filter = new WebGLImageFilter();
    filter.addFilter('hue', 180);
    filter.addFilter('negative');
    filter.addFilter('blur', 7);
    var filteredImage = filter.apply(inputImage);

**Достоинства библиотеки WebGLImageFilter:**

 - Красивый и понятный API.
 - Множество предустановленных фильтров.



### Достоинства фильтрации изображений с WebGL

 - Очень, очень, очень быстро.
 - Есть несколько хороших плагинов.
 - Можно использовать готовые шейдеры написаные на языке GLSL для OpenGL за последние 14 лет (начиная c 2001 года)
 - Возможно использовать для видео стримов.

### Недостатки фильтрации изображений с WebGL

 - Слабая поддержка браузерами.
   - Chrome, Opera - отлично, 
   - FF, Safari, IE11 - частично, 
   - все остальные - никак.
 - Для нестандартных операций порог входа достаточно высок.


Синтаксис CSS-фильтров

-webkit-filter:
  brightness(1.4)
  contrast(2)
  hue-rotate(300deg)
  sepia(0.3);
  
  
Достоинства CSS-фильтров

Быстрые.
Очень удобно описываются.

Недостатки CSS-фильтров

Пока только webkit(blink) браузеры.
Ничего кроме типовых операций. 
т.е. … Инстаграма с ними не сделаешь :(




Синтаксис

img {
    -webkit-filter: custom(
      none
      mix(
        url(someFragmentShader.fs)
        normal
        source-atop
      )
    );
}

someFragmentShader.fs

precision mediump float;
void main()
{
  css_ColorMatrix = mat4(1.0, 0.0, 0.0, 0.0,
                         0.0, 0.0, 0.0, 0.0,
                         0.0, 0.0, 0.0, 0.0,
                         0.0, 0.0, 0.0, 1.0);
}


Достоинства CSS-шейдеров

Быстрые.
Можно делать сложные нестандартные эффекты.

Недостатки CSS-шейдеров

Сложность описания, высокий порог входа.
Вообще нигде толком не поддерживаются :)