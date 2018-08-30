# Разделение корзины по поставщикам в InSales
## Суть задачи
+ На одной витрине присутствуют товары разных поставщиков. При этом нельзя чтобы в один заказ попадали товары разных поставщиков. 
+ Реализация позволяет предотвратить оформления заказов, содержащих товары от разных поставщиков. При этом надо обезпечивает возможность оформления заказов для всех товаров, которые пользователь положил в корзину.

## Принцип работы
+ Поставщик задаётся в одном из параметров карточки товара.
+ При наличии в Корзине товаров нескольких поставщиков, корзина визуально разделяется на блоки, в которых товары принадлежат только одному поставщику (имеют одинаковое значение параметра заданного в настройках темы). 
+ Блоки на странице упорядочиваются по сумме всех товаров находящихся в них (то есть вверху идёт блок с максимальной суммой).
+ Для каждого блока добавляется строка с общей суммой и кнопка Купить.
+ Имеющийся блок итоговой суммы и кнопкой купить убирается (скрывается). 
+ Перед первым блоком добавляется вывод сообщения заданного в настройках темы.
+ **При нажатии на кнопку "купить в корзине":**
  + Информация о товарах НЕ принадлежащих данному блоку (находящиеся в других блоках) сохраняются в  файлах cookies и удаляются из корзины.
  + Происходит редирект на страницу оформления заказа.
+ **На странице оформления заказа** при наличии в корзине товаров нескольких поставщиков происходит редирект на страницу корзины. 
+ **Восстановление Корзины**
  + Для того чтобы пользователь мог оформить заказы на все товары, которые были у него в корзине, реализован механизм восстановления корзины из файлов cookies. 
  + Восстановление срабатывает на всех страницах, кроме страниц чекаута.  
  + После восстановления товары видны в блоке корзины, в шапке и виджете. 

## Установка и настройка
1. В параметрах товара создаем новый параметр, по которому будем разделять товары по поставщикам. В нашем случае, этот параметр имеет название *"supplier_id"*
2. В настройки темы добавляем настроки для разделения корзины
```
<fieldset>
	<legend>Разделение корзины</legend>
	<table>
		<tr>
			<td><label for="cart_handle_property">Пермалинк параметра, для разделения корзины</label></td>
			<td><input name="cart_handle_property" id="cart_handle_property"></td>
		</tr>
      	<tr>
			<td><label for="cart_show_value">Показывать значение параметра в блоках корзины</label></td>
			<td><input name="cart_show_value" id="cart_show_value" type="checkbox"></td>
		</tr>
        <tr>
			<td><label for="cart_message">Сообщение пользователю</label></td>
			<td><input name="cart_message" id="message" type="text"></td>
		</tr>
        <tr>
			<td><label for="cart_title">Заголовок</label></td>
			<td><input name="cart_title" id="cart_title"></td>	
        </tr>
  </table>
</fieldset>
```
где *"cart_handle_property"* - название параметры из пункта #1 (а нашем случае - *"supplier_id"*)
3. Используем шаблон *"cart.liquid"* или модифицируем свой, сохраняя все настройки, скрипты и параметры из шаблона-примера
4. На все страницы добавляем 
```
	var separation_hangle = "{{ settings.cart_handle_property }}";
```

5. В форму добавления товара в корзину *<form action="/cart_items"* (страница товара и список товаров, где кнопка "в корзину"), добавить:
```
	{% assign separation_hangle = settings.cart_handle_property %}
	{% if separation_hangle.size > 0 %}
		<input type="hidden" name="comment" value="{{ product.properties[separation_hangle].characteristics.first.name }}">
	{% else %}
		{% assign separation_cart = false %}
	{% endif %}
```
							
6. На событие об изменении состава корзины (удаление позиции, или изменение количества), вешаем событие (Небходимо для изменения цены блока товаров):
```
	// Вывод цены товара в блоке, если включено разделение корзины
		var separation_cart = $('.js-separation-cart').data('separation-cart');
		var total_price_separated = [];
		if(separation_cart == true){
			// Собираем в массив цены блоков корзины
				$.each(response.order.order_lines, function(index, value) {
					if(total_price_separated[value.comment]){
						total_price_separated[value.comment] = Number(total_price_separated[value.comment]) + Number(value.total_price);
					}else{
						total_price_separated[value.comment] = Number(value.total_price);
					}
				});
						
			// Выводим новые цены в блоки корзины
				$("[data-separation-name]").each(function(index) {
					$('[data-separation-name="'+ $(this).data('separation-name') +'"]').find('.js-separated-price').html(InSales.formatMoney(total_price_separated[$(this).data('separation-name')], cv_currency_format));
				});
						
			// Скидка 
				if(response.discounts.length > 0){
					$(".cart-discounts").show();
					$(".cart-discounts strong").html(InSales.formatMoney(response.discounts[0].amount, cv_currency_format));
					$(".cart-discounts span").html(response.discount_description);
				}else{
					$(".cart-discounts").hide();
				}	
		}else{ // Если разделение корзины выключено
			// Дефолтное события
		}
```
				
7. Вешаем функцию *"reestablish_cart()"* на все страницы сайта, кроме чекаута. Вызываем ее на всех этих страницах, а так-же вешаем на на события, когда нужно очистить корзину (напр. добавление нового товара в корзину *"reestablish_cart(false)"*)
```
	<script>
		// Восстанавливаем корзину (Должно срабатывать на всех страницах, кроме чекаута)
			function reestablish_cart(refresh){
				var cookie_cart_items = $.cookie('js-separation-cart-items');
				if($.cookie('js-separation-cart') == 'true'){
					var cookie_cart_items_arr = cookie_cart_items.split(',');
					var new_cart_items_serialize;
					$.each(cookie_cart_items_arr, function(index, value){
						var cookie_cart_item = value.split(':');
						if(new_cart_items_serialize){
							new_cart_items_serialize = 'cart[quantity]['+ cookie_cart_item[0] +']='+ cookie_cart_item[1] +'&cart[order_line_comments]['+ cookie_cart_item[0] +']='+ cookie_cart_item[2] +'&'+ new_cart_items_serialize;
						}else{
							new_cart_items_serialize = 'cart[quantity]['+ cookie_cart_item[0] +']='+ cookie_cart_item[1] +'&cart[order_line_comments]['+ cookie_cart_item[0] +']='+ cookie_cart_item[2];
						}
					});

					// Добавляем товары в корзину
						$.post('/cart_items.json', new_cart_items_serialize).done(function (cart) {
							console.log(cart);
							// Удаляем эти товары из куков
								$.cookie('js-separation-cart-items', '', {
									path: '/',
									expires: 365
								});
								$.cookie('js-separation-cart', 'false', {
									path: '/',
									expires: 365
								});

							// Перезагружаем страницу
								if(refresh == true){
									location.reload();
								}
						});
				}
			}

			if(template != 'checkout'){
				reestablish_cart(true);
			}
	</script>
```
	
8. На страницу чекаута добавляем скрипт
```
	{% comment %}
		Проверка, разделена ли корзина
	{% endcomment %}
	{% assign separation_hangle = settings.cart_handle_property %}
	{% if separation_hangle.size > 0 %}
		<script>
			// Проверяем, включен ли режим разделенной корзины
			// Если включен, то проверяем, разделена ли корзина
			// Если не разделена, то отправляем в корзину
				jQuery(document).ready(function($){
					if(document.location.pathname == '/new_order'){ // Проверка, чекаут или ЛК
						if($.cookie('js-separation-cart') != 'true'){
							// Перенаправляем на страницу чекаута
								document.location.href="/cart_items";
						}
					}else{
						reestablish_cart(true);
					}
				});
		</script>
	{% endif %}
```

## Сам скрипт и пример на GitHub
https://github.com/eZ4hUNt/insdales-separation-cart

Пример: http://west.shop
