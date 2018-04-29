# Tiny Physics

## Постановка задачи: 
Симуляция взаимодействия трех тел и трех пружин между ними
## Способ реализации: 
Создание графической визуализации на языке C++ при помощи графической библиотеки SFML
## План:
   * [Создание тела и симуляция его движения](#chapter-0)
   * [Создание связи между телами](#chapter-1)
   * [Обобщение для 3 тел](#chapter-2)
   * [Обобщение для N тел](#chapter-3)
<a id="chapter-0"></a>
## Создание тела
Для упрощения расчетов, будем считать тело материальной точкой (не учитывать его форму и вращение), которая ведет себя, как центр масс. 
Тогда для описания динамики тела нам понадобится его положение, скорость и ускорение. Для их хранения будем использовать класс Vector из файла Vector.h. Создадим три соответствующих вектора: 

    FloatVector2D r (100, 100);
    FloatVector2D V (10, 10);
    FloatVector2D F (0, 0); 

Линейное движение тела описывается уравнением:
        
      r = r0 + V*dt
Где <b>r</b> - радиус-вектор положения (далее - положение), <b>r0</b> - радиус-вектор начального положения тела, <b>V</b> - его скорость, а <b>dt</b> - время, за которое это перемещение было совершено.

Однако, такой способ описания динамики тела не всегда покрывает поставленную задачу полностью, поскольку скорость тела для данной задачи не является постоянной величиной. Значение скорости при линейном ускорении описывается следующим уравнением:

      V = V0 + a*dt
Где <b>V</b> - скорость тела спустя все то же время <b>dt</b>, <b>a</b> - ускорение тела, а <b>V0</b> - начальная скорость. Второй закон Ньютона гласит:
      
      F = m*a => a = F/m
Где <b>а</b> - ускорение тела с массой <b>m</b>, а <b>F</b> - сумма сил, действующих на это тело.

Таким образом, мы получаем систему уравнений:
      
      r = V*dt + r0; (1)
      V = a*dt + V0; (2)
      a = F/m;       (3)
      
Однако становится очевидно, что эти расчеты имеют смысл лишь только если сила за все время dt не меняется. Потому мы приходим к методу численного интегрирования, которое представляет собой перерасчет всех величин каждые dt секунд. При этом, после этого <b>r0</b> станет равно <b>r </b>, a <b>V0</b> примет значение <b>V</b>. Таким образом, код на C++, отвечающий за обновление положения тела примет вид:

      r += V*dt;
      V += (F/m)*dt;
Тем не менее и здесь есть свои подводные камни. Одним из них является погрешность вычислений. Во-первых, нельзя брать dt слишком большим, поскольку изменение значения силы будет происходить крайне редко. Для наглядности, рассмотрим сразу несколько траекторий тел с разными шагами интегрирования на визуализации действия гравитации:

![dt_diff](https://github.com/kntzn/physics_springs/blob/master/img/dt_diff.png)

Отчетливо видно, как с увеличением шага интегрирования растет погрешность (удаление от истиного положения тела). 

Во-вторых, запись уравнений в таком порядке, как в примере с кодом выше (такая запись, кстати, называется явным интегрированием по Эйлеру), приведет к погрешности, поскольку обновление координат происходит до обновления скорости. Чтобы внести ясность, представим результат единичного выполнения этих двух строк:
      
      r = 0, V = 0, F = 1, m = 1;
      r += V*dt;
      V += (F/m)*dt;
      
Первая строка (Уравнение 1) не увеличит <b>r</b>, поскольку <b>V</b> равно нулю. А вот скорость, в свою очередь, возрастет на <b>dt</b>: 
      
      V += (F/m)*dt => V += 1/1*dt => V += dt

Однако простая перестановка этих двух строк исправляет ситуацию. Этот метод называется неявным интегрированием по Эйлеру. Создадим функцию обновления тела, используя этот метод:

      void updatePoint (FloatVector2D &r, FloatVector2D &V, FloatVector2D &F, 
                        const float dt, const float mass)
         {
         V += F/mass;
         r += V*dt;
         }

В качестве параметров она принимает всю информацию о теле, а так же шаг инегрирования <b>dt</b>.

Как было сказано ранее, для визуализации используется библиотека SFML. Для того, чтобы <s>кот</s> код был красив и чист, вынесем рисование нашей материальной точки в отдельную функцию, которая будет принимать объект типа sf::RenderWindow по указателю, а так же положение <b>r</b>:
        
      void drawPoint (FloatVector2D r, sf::RenderWindow &window)
        { 
        sf::CircleShape circle (30);
        circle.setPosition (r.toSf ());
        circle.setOrigin (circle.getRadius (), circle.getRadius ());

        window.draw (circle);
        }
  
Краткое пояснение: *r.toSf()* означает конвертацию из типа FloatVector2D в sf::Vector2f и эквивалентно 
  
      sf::Vector2f (r.x, r.y)
      
Таким образом, эта функция будет создавать круг, ставить его на позицию нашего тела и рисовать его на экране. 
Итого, мы имеем функцию рисования и функцию обновления положения. Если мы будем вызывать каждый раз функцию обновления тела, а потом перерисовывать его на экране, мы получим иллюзию движения. Для этого мы вызывам сначала фунцию *updatePoint*, заставляя тело перемещаться, после чего очищаем экран при помощи *window.clear()*, отрисум наше тело при помощи *drawPoint* и, наконец, отобразим все это при помощи *window.display*

Так как эти действия должны повторяться очень часто, а шаг интегрирования должен быть мал, поместим вызов этих функций в основной цикл *while (window.isOpen ())*:

    while (window.isOpen ())
      {
      window.clear ();

      updatePoint (r, V, F, dt, mass);
      drawPoint (r, window);

      window.display ();
      }
      
Запускаем и получаем ожидаемый результат. С полным кодом можно ознакомиться в папке 
[example1](https://github.com/kntzn/physics_springs/tree/master/example1)

<a id="chapter-1"></a>
## Создание связи между телами

Исходя даже из названия этого подпункта, можно сделать простой вывод: тело нам понадобится, как минимум, не одно.
Продублируем все параметры первого тела:

    FloatVector2D r1 (100, 100);
    FloatVector2D r2 (100, 200);

    FloatVector2D V1 (10, 0);
    FloatVector2D V2 (-10, 0);

    FloatVector2D F1 (0, 0);
    FloatVector2D F2 (0, 0);

    float mass1 = 1, mass2 = 2;

А так же сделаем вызов все тех же функций: *updatePoint* и *drawPoint*, но уже для тела номер 2:

      while (window.isOpen ())
        {
        window.clear ();

        updatePoint (r1, V1, F1, dt, mass1);
        updatePoint (r2, V2, F2, dt, mass2);

        drawPoint (r1, window);
        drawPoint (r2, window);

        window.display ();
        }
    
Теперь подумаем, как составить пружинную связь между телами. Согласно закону Гука,
 
    F = -k*dx

Где <b>F</b> - сила действия пружины, <b>k</b> - Коэффициент жесткости пружины, а <b>dx</b> - ее деформация.
Однако поскольку <b>F</b> - вектор, выражение *-k * dx* не должно быть скалярной величиной. Для того, чтобы использовать эту величину как вектор, домножим ее на единичный вектор разницы положений. Действительно, сила действия пружины, связывающей тела, будет направлена вдоль <b>r2 - r1</b>, что наглядно видно на картинке:

![delta_pos](https://github.com/kntzn/physics_springs/blob/master/img/delta_pos.png)

Соответственно, заведем вектор разницы положений *current_distance*:

    FloatVector2D current_distance = r2-r1;

Тогда единичныый вектор, указывающий направление от тела 1 к телу 2, будет равен:

    current_distance / current_distance.length ();
    
<a id="chapter-4"></a>
В результате, значение силы примет вид

    Force = (current_distance / current_distance.length ())*k*dx;
    
Однако в этом равенстве по прежнему фигурирует *dx*, значение которого до сих пор неизвестно. Из школьной физики мы помним, что *dx* - растяжение пружины. С другой стороны, *dx* - разница расстояний, но между чем и чем? Давайте посмотрим:

![delta_dist](https://github.com/kntzn/physics_springs/blob/master/img/delta_dist.png)

Таким образом, начальная длина пружины - это ее параметр, который где-то придется хранить. Заведем переменую, которая за это отвечает:

    float initial_distance = (r2-r1).length ();
    
Как вы видите, она равна длине вектора <b>r2-r1</b>. Действительно, мы начинаем симуляцию при условии, что пружина не растянута, следовательно начальная её длина это текущее расстояние между её концами. 

Исходя из значения *initial_distance* и длины вектора *current_distance*, можем посчитать долгожданное *dx*:

    dx = current_distance.length () - current_distance;
    
Дабы не усложнять код лишними переменными, подставим *dx* в [ранее указанную формулу](#chapter-4):

    Force = (current_distance / current_distance.length ())*(current_distance.length () - current_distance)*k;

Вспомним, куда направлен вектор силы <b>Force</b>: он сонаправлен вектору <b>r2-r1</b>. Далее, определимся со знаком: представим, что пружина растянута. Тогда *dx* - положительная величина. При этом, пружина стремится стянуть два тела (сила будет действовать на первое тело в сторону второго). Тогда *Force* будет той силой, которая действует на тело 1, откуда следует логичное равнство:

    F1 = Force;

Теперь разберемся с <b>F2</b>. Согласно третьему закону Ньютона,

    F 1->2 = -F 2->1
    
В дргуой формулировке это звучит так: *сила действия равна силе проитиводействия*, следовательно, сила действия пружины на тело 1 будет равна силе действия на тело 2 по модулю, но будет противоположна по знаку:

    F2 = -F1;
    или
    F2 = -Force;

Таким образом новым кодом внутри главного цикла станет:

    FloatVector2D current_distance = r2-r1;

    FloatVector2D Force = (current_distance/current_distance.length ()) * (current_distance.length () - initial_distance) * k;

    F1 = Force;
    F2 = -Force;

    updatePoint (r1, V1, F1, dt, mass1);
    updatePoint (r2, V2, F2, dt, mass2);
    
Запустим код и взглянем на поведение тел (точек). Все прекрасно, только вот тела двигаются с дерганием, которое обусловлено привязкой к частоте ЦП. Чтобы от этого избавиться, сделаем привязку к реальному прошедшему времени. То есть, заведем объект типа *sf::Clock*, который будет замерять прошедшее время между прохождениями цикла. Там же заведем переменную *total_dely*, которая будут аккумулировать прошедшее время и опустошаться, если времени прошло больше, чем <b>dt</b>:

    sf::Clock timer;
    float total_delay = 0;

    while (window.isOpen ())
      {
      total_delay += timer.getElapsedTime ().asSeconds ();
      timer.restart ();
      
      while (total_delay > dt)
          {
          FloatVector2D current_distance = r2-r1;

          FloatVector2D Force = (current_distance/current_distance.length ()) * (current_distance.length () - initial_distance) * k;

          F1 = Force;
          F2 = -Force;

          updatePoint (r1, V1, F1, dt, mass1);
          updatePoint (r2, V2, F2, dt, mass2);
          
          total_delay -= dt;
          }
      
      window.clear ();
      drawPoint (r1, window);
      drawPoint (r2, window);
      drawSpring (r1, r2, window);
      window.display ();
      }
      
Такая конструкция позволяет делать физические расчеты с постоянной величиной <b>dt</b> и с соответствием прошедшему времени. 

Финальной частью создания свзяи станет визуализация пружины. Для этого мы создадим объект типа *sf::RectangleShape* посередине между телами, который будем растягивать по одной из осей до длины в расстояние между телами (до длины вектора <b>r2-r1</b>). После этого мы будем поворачивать этот прямоугольник так, чтобы концами он располагался в координатах тел:

![spring_draw](https://github.com/kntzn/physics_springs/blob/master/img/spring_draw.png)

Напишем соответствующую функцию *drawSpring*, рисующую толстую прямую между точками <b>r1</b> и <b>r2</b>:

    void drawSpring (FloatVector2D r1, FloatVector2D r2, sf::RenderWindow &window)
      { 
      const float rect_length = 100;
      sf::RectangleShape rect (sf::Vector2f (rect_length, 10));
      rect.setOrigin (rect.getSize ()/2.f);
      rect.setPosition ((r1+r2).toSf ()/2.f);
      rect.setFillColor (sf::Color (127, 127, 127));

      // Rotation
      float angle = atan2f (r1.y-r2.y, r1.x-r2.x);
      angle /= pi;
      angle *= 180;
      rect.setRotation (angle);

      // Scaling
      rect.scale ((r1-r2).length ()/rect_length, 1);

      // Drawing
      window.draw (rect);
      }

С полным кодом можно ознкомиться в папке [example2](https://github.com/kntzn/physics_springs/tree/master/example2)

<a id="chapter-2"></a>
## Обобщение для 3 тел
Этот пункт является логическим продолжением предыдущего, а так же наглядным примером принципа *каждый с каждым*. То есть, каждое тело будет связано с любым другим. Поэтому, для начала, создадим третье тело и вторую пружину, чтобы получилась конструкция, напоминающая бусы: 

![2_springs](https://github.com/kntzn/physics_springs/blob/master/img/2_springs.png)

Соответственно, все параметры повторяются: создаем вектора <b>r3, V3, F3</b> для третьего тела. С пружиной чуть сложнее. Поскольку теперь уже не одна пара взаимодействует, а у каждой пружины будет своя собственная уникальная деформация, понадобится хранить не одну начальную дистанцию. Потому заведем две величины начальных дистанций:

    float initial_distance_2_3 = (r2-r3).length ();
    float initial_distance_2_1 = (r2-r1).length ();
Соответственно, заведем два вектора *current_distance_1_2* и *current_distance_2_3*:

    FloatVector2D current_distance_2_1 = r2-r1;
	 FloatVector2D current_distance_2_3 = r2-r3;
    
Так же, каждая пружина будет иметь свою собственную силу:

    FloatVector2D Force_2_1 = (current_distance_2_1/current_distance_2_1.length ()) * (current_distance_2_1.length () - initial_distance_2_1) * k;
    FloatVector2D Force_2_3 = (current_distance_2_3/current_distance_2_3.length ()) * (current_distance_2_3.length () - initial_distance_2_3) * k;

Сложнее всего решить, к чему применять эти силы. Нужно очень внимательно следить, откуда и куда будут направлены вектора. Если внимательно проследить, за индексами тех переменных, которые мы ранее заводили, то можно заметить, что первая цифра отвечает за то, к какому телу направлена сила, а вторая - за то, к какому телу она применяется. Отсюда следует, что значения сил примут следующие значения:

    F1 = Force_2_1;
    F3 = Force_2_3;

Вспомним третий закон Ньютона и приравняем силу, приложеную к телу номер 2, к сумме отрицательных значений сил F1 и F3;

    F2 = -Force_2_1 - Force_2_3;
    или
    F2 = -F1 - F3;

Аналогично введем и третью пружину.
