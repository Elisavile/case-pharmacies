## Описание задачи 

Компания «переезжает» с упрощенной системы налогообложения на общую. К вам подошел главный бухгалтер с просьбой:

В нашей сети около 100 аптек. Нам нужно единоразово загрузить файл в 1С, но для этого надо сделать сравнение реализации по аптекам и выгрузки из СБИСа. 
Пункты, которые расходятся, мы подправим вручную. После этого будем загружать в 1С. Подробная информация ниже.

* Выгрузка из СБИС находится в папке Входящие. 

* Файлы с выгрузкой из аптек находятся в папке Аптеки

## Решение задачи

1. Для начала выгружаем файлы из папки, если файл расшерения не csv, то пропускаем его и не добавляем в датафрейм. Отобрать файлы можно с помощью библиотеки `glob`, позволяющей отобрать файлы по шаблону
```
filts = glob.glob('Входящие *.csv')

dfs =[]

for file_name in filts:

  small_df = pd.read_csv(file_name, skiprows = 1, sep = ';', encoding='windows-1251', header=None)

  dfs.append(small_df)
```

2. Теперь изменим имена столбцов, имена их нескольких слов сцепим через нижнее подчеркивание
```
   name_columns = ["Дата", "Номер", "Сумма", "Статус", "Примечание","Комментарий",  

   "Контрагент", "ИНН/КПП", "Организация", "ИНН/КПП", "Тип документа", "Имя файла",

    "Дата", "Номер 1", "Сумма 1", "Сумма НДС", "Ответственный", "Подразделение",

   "Код", "Дата", "Время", "Тип пакета", "Идентификатор пакета", "Запущено в обработку",

   "Получено контрагентом", "Завершено", "Увеличение суммы", "НДC", "Уменьшение суммы", "НДС"]

   pharmacy = pd.concat(dfs, ignore_index=True)

   pharmacy.columns = name_columns

   /// сцепляем названия

   pharmacy.columns = [i.replace(' ', '_') for i in pharmacy.columns]
   
```
3. Обрабатываем файл с выгрузкой из аптек. Файл находится в определенной папке, загружаем файл по одному, если встречается файл не csv формата - пропускаем его.

```
for file in apteka:

if 'csv' not in file:

  continue

else:

  apts = pd.read_csv('Аптеки/correct/csv/' + file, sep=';', encoding='windows-1251')
```

4. Добавляем недостающие столбцы

```
   apts['Номер счет-фактуры'] = ""
  
   apts['Сумма счет-фактуры'] = ""
   
    apts['Дата счет-фактуры'] = ""
   
    apts['Сравнение дат'] = ""
```
5. В каждой строке проверяем условия:

* Если "Поставщик" - ЕАПТЕКА, то к "Номер накладной" добавляем /15.
* Нужно найти все записи в выгрузке из СБИСа по данному номеру накладной
* Из найденных строк нужно оставляем только те, которые имеют один из типов документа: ["СчФктр", "УпдДоп", "УпдСчфДоп", "ЭДОНакл"]. Если ничего не найдено - просто переходим к следующей строке. Если найдено - сохраняем значения Номер, Сумма и Дата
* Дату нужно представить в формате 25.05.2021
* В столбцы из пункта 6 нужно записать найденные для данной строки значения
* В столбец Сравнение дат помещаем "Не совпадает!", если найденная дата и дата накладной отличаются. Иначе - пустая строка.
```
type_doc = ["СчФктр", "УпдДоп", "УпдСчфДоп", "ЭДОНакл"]
    for i, row in apts.iterrows():
        name_note = row["Номер накладной"]

        if 'ЕАПТЕКА OOO' in row["Поставщик"]:
            name_note += "/15"

        writ = pharmacy[pharmacy['Номер'] == name_note]
        writ = writ[writ['Тип_документа'].isin(type_doc)]

        if writ.empty:
            continue

        invoice = writ.iloc[0]["Номер"]
        summ = writ.iloc[0]["Сумма"]
        date = writ.iloc[0]["Дата"][1]
        date = datetime.strptime(date, "%d.%m.%y").strftime("%d.%m.%Y")

        apts.at[i, "Номер счет-фактуры"] = invoice
        apts.at[i, "Сумма счет-фактуры"] = summ
        apts.at[i, "Дата счет-фактуры"] = date

        if (date == apts.at[i, 'Дата накладной']):
          apts.at[i, "Сравнение дат"] = ""
        else:
          "Не совпадает!"
```
6. Меняем имена столбцов
```
   apts_columns = ['№ п/п', 'Штрих-код партии', 'Наименование товара', 'Поставщик',
    'Дата приходного документа', 'Номер приходного документа',
    'Дата накладной', 'Номер накладной', 'Номер счет-фактуры',
    'Сумма счет-фактуры', 'Кол-во',
    'Сумма в закупочных ценах без НДС', 'Ставка НДС поставщика',
    'Сумма НДС', 'Сумма в закупочных ценах с НДС', 'Дата счет-фактуры', 'Сравнение дат']

   apts.columns = apts_columns
```
7. Расчитываем столбцы  "Сумма НДС" и "Сумма в закупочных ценах с НДС"  
```
  apts['Сумма НДС'] = apts['Сумма в закупочных ценах без НДС'] * apts['Ставка НДС поставщика']

  apts['Сумма в закупочных ценах с НДС'] = apts['Сумма НДС'] + apts['Сумма в закупочных ценах без НДС']
```
8. Файл сохраняем по пути Результат/{полная сегодняшняя дата}/{имя исходного файла без расширения} - результат.xlsx. Если таких папок не существует - создаем их
   
```
   apts.to_excel(f"{file.split('.csv')[0]} - результат.xlsx", index=False, encoding="windows-1251")
```
9. Проводим ABC-анализ. Для начала группируем данные по поставщикам и считаем количество товаров поставляемые каждым поставщиком
```
    post = apts.groupby(['Поставщик']).agg({'Наименование товара' : 'count'}).reset_index()
    post.columns = ['Поставщик', 'Количество поставок товара']
    post = post.sort_values(['Количество поставок товара'], ascending=False)

    post['Доля в общих поставках'] = post['Количество поставок товара'] /  sum(post['Количество поставок товара'])
    post['Накопленное количество поставок'] = post['Доля в общих поставках'].cumsum()
    post['ABC_анализ'] = np.where(post['Накопленное количество поставок']< 0.8, 'A', np.where(post['Накопленное количество поставок'] < 0.95,'B', 'C'))
```
10. Посчитаем количества значений ы каждой группе    
```
    post['ABC_анализ'].value_counts()
```
## Выводы

* A — лидеры: 20% поставщики, которые приносят 80% поставок от общего объема.
* B — середнячки: 30%, которые приносят 15% поставок от обзего объема поставок
* C — аутсайдеры: оставшиеся 50%, которые составляют 5% поставок
  
Группа А: Высокое количество поставок. Это обозначает, что эти продукты/поставки являются наиболее значимыми для бизнеса и часто используются. Управление этими поставками должно быть оптимизировано для обеспечения непрерывности поставок и эффективного управления запасами.

Группа B: Среднее количество поставок. Эти поставки имеют менее высокую значимость по сравнению с товарами/поставками из группы A, но все равно они важны для бизнеса. Управление этими поставками должно быть более внимательным, чем для товаров/поставок из группы C.

Группа C: Низкое количество поставок. Эти поставки обычно имеют низкую значимость и не используются так часто, как товары из групп A и B. Управление этими поставками может включать в себя более гибкие стратегии и может потребовать меньше внимания и ресурсов.

Исходя из этого, можно сделать вывод, что для эффективного управления поставками необходимо сконцентрировать большую часть внимания и ресурсов на тех товарах/поставках, которые входят в группу A, а также обеспечить оптимальное управление товарами/поставками из группы B и C в соответствии с их значимостью.
