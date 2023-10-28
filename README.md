if (!window["App"]) throw new Error("Critical Error in Application");
(function m(App) {


    //===========================================================================================================


    function lib() {
        var _windowContainer = null,
        _guid = null,
        _poll = this,
        _name = '',
        _store = null;

        var header, panel;

        var save = function(){ //Функция сохранения изменений в анкете
            if (!panel.container['form'].pharmId) return false;

            var modifyData = '';

            $.each(_store.getModifiedRecords(), function(index, rec){
                $.each(rec.modified, function(name){
                    modifyData += '&values['+index+']['+name+']=' + rec.data[name];
                });
                modifyData += '&id['+index+']=' + rec.id;
            });
            if (modifyData == '') return;
                $.getJSON('./modules/units/units.php?action=SET_FORMDATA&pharmId='+panel.container['form'].pharmId+'&projectId='+_param.projectId+'&stageId='+_param.stageId + modifyData +'&rnd='+Math.random(), function(data){
                if (!data || !data.result){
                    alert('ошибка сохранения, попробуйте еще раз');
                    //_store.rejectChanges();
                }else if (data.result == true){
                    _store.commitChanges();
                    panel.container['form'].loadData(panel.container['form'].pharmId);
                }
                $.unblockUI();
            }.bind(this));
        }

        var Header = function (container) {
            var id = 'li[rel = "' + _guid + '"] '+ ".context";
            var template = '<div class="header" style="clear:both; font-size: small;"><div class="context" style="clear:both; overflow:hidden"></div><div class="header_button collapse"></div><div class="buttons">&nbsp;</div></div>';
            Header.superClass.apply(this, [id, template, container]);

            this.appendTo = function (parent) {
                $(parent).append(this.template);
                return this;
            }

            var _expanded = true;

            this.expand = function(callback){
                _expanded = true;
                _windowContainer.find('.header_button').removeClass('expand').addClass('collapse');
                this.element().animate({
                    height: this.container['title'].element().outerHeight() + this.container['description'].element().outerHeight()
                }, "fast", callback);
            }

            this.height = function (height) {
                return _windowContainer.find('.header').outerHeight()
            }

            this.expanded = function(){
                return _expanded;
            }

            this.addButton = function(caption, delegate){
                _windowContainer.find('.buttons').append('<input type="button" id="'+caption+'" value="'+caption+'" />');
                _windowContainer.find('.buttons #'+caption).click(delegate);
            }

            this.collapse = function(callback){
                _expanded = false;
                _windowContainer.find('.header_button').removeClass('collapse').addClass('expand');
                this.element().animate({
                    height: this.container['title'].element().outerHeight()
                }, "fast", callback);
            }

            this.toggle = function(callback){
                if (_expanded){
                    this.collapse(callback);
                }else{
                    this.expand(callback);
                }
            }

            this.click = function(delegate){
                if (!delegate){
                    _windowContainer.find('.header_button').click();
                }else{
                    _windowContainer.find('.header_button').unbind().click(delegate);
                }
            }
        }
        Header.inherits(App.VisualElement);//Наследуем элемент от базового из файла part_of_core.js

        var Title = function(){
            var id = 'li[rel = "' + _guid + '"] '+ ".title";
            var template = '<div class="title" style="clear:both">Приближая весну</div>';
            Title.superClass.apply(this, [id, template]);
        }
        Title.inherits(App.VisualElement);
        
        var Description = function(){
            var id = 'li[rel = "' + _guid + '"] '+ ".description";
            var template = '<div class="description" style="clear:both"></div>';
            Description.superClass.apply(this, [id, template]);
        }
        Description.inherits(App.VisualElement);
                
        var Panel = function(container){
            var id = 'li[rel = "' + _guid + '"] '+ ".panel";
            var template = '<div class="panel" style="width: 100%;"></div>';
            Panel.superClass.apply(this, [id, template, container]);
        }
        Panel.inherits(App.VisualElement);

        var Grid = function () { 
            var id = 'li[rel = "' + _guid + '"] '+ ".grid";
            var template = '<div class="grid" style="background-color: #98fb98; float: left; width: 250px;"></div>';
            Grid.superClass.apply(this, [id, template]);

            var store = new Ext.data.JsonStore({// аналог "DataTable" из состава ExtJS
                root: 'units',
                totalProperty: 'count',
                idProperty: 'id',
                remoteSort: false,

                fields: [
                    'id', 'name', 'address', 'town', {name: 'checked', type: 'bool'}
                ],

                url: 'modules/units/units.php?action=GET_ACTIVE_PHARMACIES&projectId='+_param.projectId+'&stageId='+_param.stageId+'&rnd='+Math.random()
            });
            store.setDefaultSort('id', 'desc');

            function renderTopic(value, p, record) {
                return String.format('{1}, {2} {3}', record.data.town, value, record.data.address);
            }

            var grid = new Ext.grid.GridPanel({//многофункциональный грид из состава ExtJS
                store: store,
                trackMouseOver: false,
                disableSelection: false,
                width: 250,
                loadMask: new Ext.LoadMask(Ext.getBody(), {onLoad: function(){ $.unblockUI(); } , onBeforeLoad: function(){ $.blockUI(); } }),

                columns: [
                {
                    header: "Аптека",
                    dataIndex: 'name',
                    width: 200,
                    renderer: renderTopic,
                    sortable: true
                }],

                viewConfig: {
                    forceFit: true,
                    enableRowBody: true,
                    showPreview: true
                }
            });

            this.rowClick = function(event){
                grid.on('rowclick', function(grid, rowIndex, e) {
                    var r = grid.getStore().getAt(rowIndex);
                    event(r, rowIndex);
	            });
            }            

            this.height = function (height) {
                grid.setHeight(height);
            }

            this.appendTo = function (parent) {
                $(parent).append(this.template);

                store.load({
                    params: { rnd: Math.random() },
                    callback: function () {
                        $(_poll).trigger({ type: "poll", event: "complete" });
                    } .bind(this)
                });

                grid.render(Ext.select(this.id, true));
            }
        }
        Grid.inherits(App.VisualElement);

        var Form = function(){
            var id = 'li[rel = "' + _guid + '"] '+ ".form";
            var template = '<div class="form" style="margin: 0 auto;"></div>';
            Form.superClass.apply(this, [id, template]);
            
            var grid, height;

            this.pharmId = null;

            var config = {
                forceFit: true,
                enableRowBody: false
            };

            this.appendTo = function (parent) {
                $(parent).append(this.template);
                loadHeader.call(this);
            }

            this.height = function (newHeight) {
                height = newHeight;
                if (grid){
                    grid.setHeight(height);
                }
            }

            var loadHeader = function(){ //Шапка анкеты приходит из базы, но только один раз
                $.getJSON('./modules/units/units.php?action=GET_FORMHEADER&projectId=' + _param.projectId + '&stageId=' + _param.stageId + '&rnd='+Math.random(), function(data){
                    var mask = new Ext.LoadMask(Ext.getBody(), {onLoad: function(){ $.unblockUI(); } , onBeforeLoad: function(){ $.blockUI(); } });                    

                    _store = new Ext.data.GroupingStore({
                        reader: new Ext.data.JsonReader({fields: Ext.data.Record.create(data.fields)})
                    });

                    if ($.isArray(data.groups)){
                        var group = new Ext.ux.grid.ColumnHeaderGroup({
                            rows: data.groups
                        });
                    }

                    var proxy = new Ext.data.HttpProxy(
			        {
			            url: './modules/units/units.php?action=GET_LIST&rnd'+Math.random(),
			            method: 'POST'
			        });                  

                    for (var i = 0; i < data.columns.length; i++){
                        if (data.columns[i].editor && data.columns[i].editor.xtype == 'combo'){
                            data.columns[i].width = 130;
                            var store = new Ext.data.JsonStore(
                            {
                                fields: ['rowId','name'],
                                proxy: proxy,
                                baseParams: { listId: data.columns[i].editor.listId },
                                idProperty: 'rowId',
                                root: 'records',
                                autoLoad: true
                            });
                            data.columns[i].editor =  { 
                                xtype: 'combo',
                                typeAhead: true,
                                lazyRender: true,
                                triggerAction: 'all',
                                store: store,
                                mode: 'local', 
                                displayField: 'name',
                                valueField: 'rowId',
                                editable: false,
                                selectOnFocus:true
                            }
                        }
                    }

                    grid = new Ext.grid.EditorGridPanel({ //Грид анкеты
                        store: _store,
                        columns: data.columns,
                        width: 'auto',
                        height: height,
                        loadMask: mask,
                        clicksToEdit: 0,
                        viewConfig: config,
                        plugins: ($.isArray(data.groups)? group : undefined)
                    });

                    grid.render(Ext.select(this.id, true));
                }.bind(this));
            }

            this.loadData = function(pharmId){ //Загрузка данных анкеты в созданую таблицу (Form)
                this.pharmId = pharmId;
                _store.rejectChanges();
                $.getJSON('./modules/units/units.php?action=GET_FORMDATA&pharmId=' + pharmId + '&projectId=' + _param.projectId + '&stageId=' + _param.stageId + '&rnd='+Math.random(), function(data){
                    _store.loadData(data.rows);
                    $.unblockUI();
                });
            }
        }
        Form.inherits(App.VisualElement);

        //===========================================================================================================


        function main(viewPort) { //создается окно модуля
            _windowContainer = viewPort;

            header = new Header(); //шапка модуля (документа)
            panel = new Panel();

            header.appendTo(_windowContainer.find('.body')); //добавляем ее на рабочую область
            header.add({
                title: new Title(),
                description: new Description()
            }); //Добавляем на шапку рабочие элементы

            header.container['description'].element().load('./modules/units/forms.html?'+Math.random(), function(){
                $.unblockUI();
                panel.container['grid'].height($('#window').outerHeight() - header.height());
                panel.container['form'].height($('#window').outerHeight() - header.height());
            }.bind(this)); //загружаем "статический" текст-описание

            header.addButton('save', function(){
                save();
            }); //обработчик на кнопку сохранения

            panel.appendTo(_windowContainer.find('.body')); //Панель эта главная рабочая область окна
            panel.add({
                grid: new Grid(),
                form: new Form()
            }); //На нее добавляем список аптек и таблицу - анкету

            panel.container['grid'].rowClick(function(record, index){
                panel.container['form'].loadData(record.id);
            }); //При выборе аптеки подгружаем анкету

            header.click(function(){
                panel.hide();
                header.toggle(function(){
                    panel.show();                    
                    panel.container['grid'].height($('#window').outerHeight() - header.height());
                    panel.container['form'].height($('#window').outerHeight() - header.height());
                });
            }); //При нажатии на шапку - сворачиваем ее
        }

        var sync = function (message) {
            switch (message.event) {//сообщения приходят только от ядра, 
                                    //если другой модуль хочет отправить сообщение этому модулю
                                    //то все равно его "прокидывает" ядро по таргету
                case "binding": //сообщение приходит один раз при успешной привязке к ядру
                    $(window.document).unbind('ajaxStart');
                    _guid = message.guid;
                    _name = message.name;
                    _param = message.data;
                    if (_param.edit == undefined) {
                        _param.edit = true
                    }
                    main(message.viewPort);
                    break;
                case "unbinding": //сообщение приходит один раз при выгрузке модуля
                    $(window.document).ajaxStart($.blockUI);
                    break;
                case "show": //сообщение приходит всякий раз когда модуль становится "активным", 
                             //т.е. окно модуля получает фокус
                    $(window.document).unbind('ajaxStart');
                    break;
                case "hide": //сообщение приходит при скрытии окна 
                    $(window.document).ajaxStart($.blockUI);
                    break;
                case "message": //любое другое сообщение, например служебное или от другого модуля
                    alert(message.body.test);
                    break;
            }
        };

        this.entry = function (message) {
            sync(message); //Точка входа в модуль
        };

        this.getGuid = function () { //каждый экземпляр модуля получает уникальный id, 
                                     //на один модуль может быть много экземпляров - в аналогии с ООП
                                     //модуль это класс, экземпляр модуля это объект
            return _guid;
        };

        this.getName = function () {
            return _name;
        };
    }


    //===========================================================================================================

    App.Core.Registry(lib, "units_forms"); //Регистрация модуля в ядре
})(App);
part_of_core.js
(function a(appSpace) {


    //===========================================================================================================

<cute>

    //===========================================================================================================


    var BaseList = function (typeOfList, keeper, collect) { //Класс для работы с коллекцией документов
        var collection = [];
        this.keeper = keeper;
        if (!typeOfList) throw new Error("Type is not response");
        if (!keeper) throw new Error("Keeper is not response");

        this.Add = function (elements) {
            if ($.isArray(elements)) {
                for (var i = 0; i < elements.length; i++) {
                    if (!elements[i].id) {
                        throw new Error("Element index: " + i + " is not have id property");
                    }
                    if (collection[elements[i].id]) {
                        throw new Error("Element with id: " + elements[i].id + " is already exists");
                    }
                    collection[elements[i].id] = new typeOfList(elements[i], this);
                }
            } else if ($.isPlainObject(elements)) {
                if (!elements.id) {
                    throw new Error("Element is not have id property");
                }
                if (collection[elements.id]) {
                    throw new Error("Element with id: " + elements.id + " is already exists");
                }
                collection[elements.id] = new typeOfList(elements);
            } else {
                throw new Error('Error type: must be object or array');
            }
            return this.List();
        };

        this.Clear = function () {
            collection = [];
            return this.List();
        };

        this.Remove = function (id) {
            collection[id] = null;
            collection.remove(id);
            return this.List();
        };

        this.List = function (filter) {
            return collection;
        };

        this.Get = function (id) {
            return collection[id];
        };

        this.Add(collect);
        return this.List();
    };


    //===========================================================================================================


    var VisualElement = function (id, template, container) { //Базовый класс для всех визуальных элементов системы
        this.id = id;
        this.template = $(template);
        this.container = container || {};

        this.hide = function () {
            $(this.id).hide();
            return this;
        }

        this.show = function () {
            $(this.id).show();
            return this;
        }

        this.element = function(element){
            if (!element){
                return $(this.id);
            }else{
                return $(this.id).find(element);
            }
        }

        this.isVisible = function () {
            return $(this.id).is(":visible");
        }

        this.resize = function (width, height) {
            for (var el in container) {
                this.container[el].resize(width, height);
            }
        }

        this.height = function (height) {
            return $(this.id).height(height);
        }

        this.width = function (width) {
            return $(this.id).width(width);
        }

        this.empty = function () {
            $(this.id).empty();
            this.container = {};
            return this;
        }

        this.click = function (delegate) {
            if (!delegate) {
                $(this.id).click();
            } else {
                $(this.id).unbind().click(delegate);
            }
        }

        this.toggle = function (actionOne, actionTwo) {
            if (!actionOne) {
                $(this.id).toggle();
            } else {
                $(this.id).unbind().toggle(actionOne, actionTwo);
            }
        }

        this.add = function (component) {
            for (var el in component) {
                this.container[el] = component[el];
                if (!(this.container[el] instanceof VisualElement)) throw "container is not Element type";
                this.container[el].appendTo(this.id);
            }
        }

        this.appendTo = function (parent) {
            if (template != undefined){
                $(parent).append(this.template)
            }else{
                $(parent).append(this.id)
            }
            return this;
        }
    }


    //===========================================================================================================

<cute>

    //===========================================================================================================


    var Core = function () {
        var isReady = false,
        timer = [],
        modules = {},
        libraries = {},
        mainInterface = null;

        var libraryList = function (libList){ //Загрузка списка библиотек доступных системе
            $.each(libList, function (index, lib) {
                libraries[lib.libraryName] = lib;
                lib.pageConstructor = App[lib.pageConstructor];
                lib.status = 'waiting';
            });
        }

        var loadLibrary = function (libName, callback) {//Загрузка конкретной библиотеки
            var lib = libraries[libName];

            if (!lib) throw new Error("Library not found");
            if (lib.status == 'removed') throw new Error("Library is removed");
            if (lib.status == 'waiting') {
                Utils.LoadJS([{ library: lib.libraryPath, unload: true, GC: []}], callback);
            }else if (lib.status == 'loaded'){
                if ($.isFunction(callback)) {
                    callback();
                }
            }
        }

        var instanceModule = function (moduleName, caption, data, pin, libName, callback) { //Инстанцирование экземпляра (модуля) библиотеки
            if (!libName) return;
            var module = null;
            var guid = null;
            $.each(modules, function(index, item){ //Ищем, вдруг модуль с таким именем уже создан
                if (item.name == moduleName){
                    module = item;
                    guid = index;
                    return false;
                }
            });

            loadLibrary(libName, function(){
                var lib = libraries[libName];

                if (!module){ //Если модуль создан пропускаем добавление его в систему
                    guid = Utils.GUID();
                    modules[guid] = {
                        name: moduleName,
                        guid: guid,
                        libraryName: libName,
                        moduleRef: null
                    };
                    module = modules[guid];

                    var page = new lib.pageConstructor(guid, caption); //Создаем окно модуля по "шаблону"
                    module.status = 'loading';
                    Utils.LoadHTML(page, lib.view, function (viewPort) { //Загружаем HTML который будем выступать для модуля контекстом
                        Utils.LoadCSS(lib.css, guid); //подгружаем необходимые библиотеки CSS
                        Utils.LoadJS(lib.request, function () { //подгружаем необходимые библиотеки JS (зависимости)
                            $.each(lib.request, function (index, item) {
                                $('script[id="' + item.libraryPath + '"]').addClass(guid); //Создаем ссылки нашего модуля на библиотеку, чтобы можно было отслеживать когда никто из модулей не использует библиотеку, и ее можно удалить из системы
                            });
                            module.moduleRef = new lib.classRef(); //Инстанцирование экземпляра
                            $(module.moduleRef).bind("poll", sync); //Сообщаем загруженному экземпляру ссылку на метод общения с ядром
                            module.moduleRef.entry({ event: "binding", guid: guid, viewPort: viewPort, name: moduleName, data: data }); //Сообщаем ядру его GUID и главный контекст
                            if ($.isFunction(callback)) {
                                callback(module);
                            }
                        });
                    });
                }else{
                    if (module.status == 'complete'){ 
                        if ($.isFunction(callback)) {
                            callback(module);
                        }
                    }else if(module.status == 'registered'){ //Если модуль загружен, но у него отсутствует контекст
                        var page = new lib.pageConstructor(guid, caption);
                        module.status = 'loading';
                        Utils.LoadHTML(page, lib.view, function (viewPort) {
                            module.moduleRef.entry({ event: "binding", guid: guid, viewPort: viewPort, name: moduleName, data: data });
                            if ($.isFunction(callback)) {
                                callback(module);
                            }
                        });
                    }
                }
            });
        };

        var removeModule = function (guid) { //Удаление модуля и чистка библиотек если нужно
            var lib = libraries[modules[guid].libraryName];
            var selfAndRequest = [{ library: lib.libraryPath, unload: true, GC: []}];
            selfAndRequest = selfAndRequest.concat(lib.request);
            var success = Utils.UnloadJS(selfAndRequest, guid);
            if (success) {
                $.each(lib.request, function (index, item) {
                    if (item.unload) {
                        for (var i = 0; i < item.GC.length; i++) {
                            window[item.GC[i]] = null;
                        }
                    }
                });
            }
            Utils.UnloadCSS(lib.css, guid);
            delete modules[guid];
        };

        var sync = function (data) { //Точка входа для сообщений модулей
            switch (data.event) {
<cute>
            }
        };

        var ready = function (func) {
            if (isReady) {
                return func();
            } else {
                var id = (new Date()).getTime();
                timer[id] = setInterval(function () {
                    if (isReady) {
                        clearInterval(timer[id]);
                        func();
                    }
                } .bind(this), 13);
            }
        };

        var login = function(){ //окно логина
            $.getJSON('./modules/common/login.php?action=CHECK&rnd='+Math.random(), function(data){
                if (data.result == true){
                    run({userName: data.userName});
                }else{
                    $('#site-wrapper').load('./modules/common/login.html?'+Math.random(), function (response, status, xhr) {
                        if (status == "success") {
                            $.each(modules, function (index, module) {
                                removeModule(module.guid);
                            });

                            $('input#in').click(function(){
                                $.getJSON('./modules/common/login.php?action=SEND&user=' + $('input#login').val() + '&pwd=' + $('input#pwd').val() + '&rnd='+Math.random(), function(data){
                                    if (data.result == true){
                                        run({userName: data.userName});
                                    }else{
                                        $('input#pwd').val('');
                                    }
                                    $.unblockUI();
                                });
                                return false;
                            });
                        } else {
                            throw new ErrorEx(status, "Sorry but there was an error: " + xhr.status + " " + xhr.statusText);
                        }
                        $.unblockUI();
                    });
                }
            });
        };

        var run = function(data){ //загрузка приложения
            $('#site-wrapper').load('./modules/common/template.html?'+Math.random(), function (response, status, xhr) {
                if (status == "success") {              
                    //====================================================================================
                    mainInterface = new MainInterface({
                        header: new Header(),
                        center: new Center()
                    });
                    mainInterface.appendTo("body");
                    mainInterface.container['header'].add({
                        windowList: new WindowList(),
                        logout: new Logout()
                    });
                    mainInterface.container['center'].add({
                        menu: new RightMenu(),
                        windows: new Windows()
                    });
                    mainInterface.container['header'].container['logout'].onClick(function(){
                        $.getJSON('./modules/common/login.php?action=LOGOUT&rnd='+Math.random(), function(data){
                            if (data.result == true){
                                login();
                            }
                            $.unblockUI();
                        });
                    });
                    //====================================================================================
<cute>
                    //====================================================================================
                    isReady = true;

                    $.getJSON('./core/modules.txt?'+Math.random(), function(data){
                        libraryList(data);
                        $.getJSON('./modules/common/login.php?action=MENU&rnd='+Math.random(), function(data){ //
                            mainInterface.container['center'].container['menu'].setItems(data);
                            $.unblockUI();
                        });
                    });

                    mainInterface.container['header'].userInfo(data.userName);
                } else {
                    throw new ErrorEx(status, "Sorry but there was an error: " + xhr.status + " " + xhr.statusText);
                }
                $.unblockUI();
            } .bind(this));
        }

        return {
            Ready: function (func) {
                ready(func);
            },

            Registry: function (libraryRef, libraryName) { //Метод регистрации для внешних модулей
                var lib = libraries[libraryName];
                if (lib.status == 'waiting'){
                    lib.classRef = libraryRef;
                    lib.status = 'loaded';
                }
            },

            Run: function () { //Запуск приложения
                $(window.document).ajaxStart($.blockUI);//.ajaxStop($.unblockUI);
                login();
            }
        };
    } ();


    //===========================================================================================================

<cute>

    //===========================================================================================================


    appSpace = Utils.Namespace(appSpace);
    appSpace["Core"] = Core;
    appSpace["BaseList"] = BaseList;
    appSpace["VisualElement"] = VisualElement;
    appSpace["Scrollable"] = Scrollable;
    appSpace["AsDocument"] = AsDocument;
})("App");
units.php
<?php

error_reporting(0);

require "../../lib/config.php";

session_start(); 
$p_id = $_SESSION['userId'];

$action = $_REQUEST['action'];

//В этом файле особо описывать нечего, ключевые моменты:
//1. Для избежания инъекций используется bind параметров к запросам и процедурам
//2. Все данные конвертируется в JSON путем сериализации объекта PHP
//3. Id текущего пользователя хранится в сессии, и сохраняется при входе в систему ($p_id)
//4. Действие GET_FILLING выводит обычную html таблицу в связи с большим объемом данных и слабыми целевыми машинами
//5. Большая часть логики приложения выделена в СУБД, PHP выступает в качестве сервиса получения JSON данных из базы.
//Не считаю целесообразным для таких целей использовать в PHP ООП подход, т.к. каждое обращение к серверу является 
//атомарным, и разворачивание большого количества объектов на один запрос пустая трата ресурсов сервера.
//ООП в PHP целесообразно использовать в случае частого использования кода в разных модулях, например: подключение к 
//базе, работа с какими-то форматами данных, система маршрутов (routes) и тд.
switch($action){
	case 'GET_PHARMACIES':
		$obj = array("units" => array());
		
		$employeeId = $_REQUEST['employeeId'];
		$employeeId = $employeeId > 0 ? $employeeId : $p_id;
		
		$result = $db_connection->prepare("call filter_userPoints(?);");
		$result->bind_param('i', $employeeId);
		$result->execute();

		$result = $db_connection->query("select p.id, p.name, p.address, p.town, if(pr.id is null, 0, 1) as checked, pr.approved from 
									t_filteredPoints t 
									inner join points p on p.id=t.id 
									left outer join projects_registeredPoints pr on pr.pointsId=t.id limit 200;");
		$obj['count'] = $result->num_rows;	
		while ($row = $result->fetch_array()) 
		{
			array_push($obj['units'], array(id => $row['id'], name => $row['name']));
			$obj['units'][count($obj['units'])-1]['address'] = $row['address'];
			$obj['units'][count($obj['units'])-1]['town'] = $row['town'];
			$obj['units'][count($obj['units'])-1]['checked'] = $row['checked'];
			$obj['units'][count($obj['units'])-1]['approved'] = $row['approved'];
		}
		$result->free();
		
		break;

	case 'GET_ACTIVE_PHARMACIES':
		$p_projectId = $_REQUEST['projectId'];
		$p_stageId = $_REQUEST['stageId'];
		$employeeId = $_REQUEST['employeeId'];
		$employeeId = $employeeId > 0 ? $employeeId : $p_id;
		
		$obj = array("units" => array());
		
		$result = $db_connection->prepare("call filter_userPoints(?);");
		$result->bind_param('i', $employeeId);
		$result->execute();

		$result = $db_connection->prepare("select p.id, p.name, p.address, p.town from 
									t_filteredPoints t 
									inner join points p on p.id=t.id 
									inner join projects_registeredPoints pr on pr.approved = 1 and pr.pointsId=t.id and pr.projectsId = ? and (pr.projectStagesId = ? OR pr.projectStagesId is NULL) order by p.town, p.name limit 200;");
		$result->bind_param('ii', $p_projectId, $p_stageId);
		$result->execute();
		
		$result->bind_result($id, $name, $address, $town);

		$count = 0;
		while ($result->fetch()) {
			array_push($obj['units'], array(id => $id, name => $name));
			$obj['units'][count($obj['units'])-1]['address'] = $address;
			$obj['units'][count($obj['units'])-1]['town'] = $town;		
			$count++;
		}
		$obj['count'] = $count;
		
		$result->close();
		
		break;

	case 'REGISTER_PHARMACY':
		$pharm = $_REQUEST['pharmId'];
		$project = $_REQUEST['projectId'];
		$registered = $_REQUEST['registered'];
		if($registered==1)
		{
			$sql = "insert into projects_registeredPoints (projectsId, pointsId) values (?, ?);";
		} else {
			$sql = "delete from projects_registeredPoints where projectsId=? and pointsId=?;";
		}
		$result = $db_connection->prepare($sql);
		$result->bind_param('ii', $project, $pharm);
		$result->execute();
		$result->close();

		$obj = new stdClass();
		$obj->result = true;
		
		break;

	case 'GET_FORMDATA':
		$projectId = $_REQUEST['projectId'];
		$stageId = $_REQUEST['stageId'];
		$pharmId = $_REQUEST['pharmId'];
		
		$sql = "call data_getFormRows (?, ?, ?);";
		$result = $db_connection->prepare($sql);
		$result->bind_param('iii', $projectId, $stageId, $pharmId);
		$result->execute();
		
		$result->bind_result($id, $name, $field, $value);

		$obj = new stdClass();
		$prev = 0;
		$element = array();
		$obj->rows = array();
		
		while ($result->fetch()) {
			if (count($obj->rows) == 0 || $obj->rows[count($obj->rows)-1]->id != $id){
				array_push($obj->rows, new stdClass()); 
				$obj->rows[count($obj->rows)-1]->id = $id;
				$obj->rows[count($obj->rows)-1]->name = $name;	
			}
			$obj->rows[count($obj->rows)-1]->{$field} = $value;
		}
		
		$result->close();
		
		break;

	case 'SET_FORMDATA':
		$projectId = $_REQUEST['projectId'];
		$stageId = $_REQUEST['stageId'];
		$pharmId = $_REQUEST['pharmId'];
		$values = $_REQUEST['values'];
		$KpiId = $_REQUEST['id'];
		$obj = new stdClass();
		
		$db_connection->query("START TRANSACTION;");
		
		for ($i = 0; $i < count($KpiId); $i++){
			foreach ($values[$i] as $key => $value){
				$sql = "select data_setNumericField(?, ?, ?, ?, ?, ?, ?);";
				$kpi2stage = str_replace('f_', '', $key);			
				$result = $db_connection->prepare($sql);
				$result->bind_param('iiiiiii', $p_id, $projectId, $stageId, $pharmId, $kpi2stage, $KpiId[$i], $value);
				$result->execute();
				$result->bind_result($headId);
				$result->fetch();

				$result->close();

				$obj->result = (isset($headId) && $headId > 0);
			}
		}
		
		if ($obj->result == true){
			$db_connection->query("COMMIT;");
		}else{
			$db_connection->query("ROLLBACK;");
		}
		
		$force = 0;
		$result = $db_connection->prepare("call formula_recalcProjectsStagePoint(?, ?, ?, ?)");
		$result->bind_param('iiii', $pharmId, $projectId, $stageId, $force);
		$result->execute();

		$result->close();
		
		break;

	case 'GET_FILLING':
		$projectId = $_REQUEST['projectId'];
		$userTypeId = $_SESSION['userTypeId'];
		$employeeId = $_REQUEST['employeeId'];
		$employeeId = $employeeId > 0 ? $employeeId : $p_id;
		$out = '';
		$stages = array();
		$rowNames = array();

		$rows = array();

		$result = $db_connection->prepare("call fillReport_getStageRows(?)");
		$result->bind_param('i', $employeeId);
		$result->execute();
		$result->bind_result($rowName);
		while($result->fetch())
		{
			array_push($rowNames, $rowName);
		}
		$result->close();
		while($db_connection->next_result()) $db_connection->store_result();

		$rows[0] = array();
		array_push($rows[0], count($rowNames));
		array_push($rows[0], 'Сотрудники');
		$rows[1] = array();
		$result = $db_connection->prepare("select a.id, concat(a.name, ' (', DATE_FORMAT(a.startDate, '%d.%m.%Y'), ' - ', DATE_FORMAT(a.endDate, '%d.%m.%Y'), ')') as name from projects_stages a INNER JOIN projects_stageUserTypes tt on tt.projectStagesId = a.id and (tt.userTypesId = (SELECT userTypesId FROM users WHERE id = ?) OR (tt.userTypesId = 1 AND (SELECT userTypesId FROM users WHERE id = ?) IN (3, 4, 6)) OR (SELECT userTypesId FROM users WHERE id = ?) IN (5)) where a.projectsId=? and DATEDIFF(NOW(), a.startDate)>=0");
		$result->bind_param('iiii', $employeeId, $employeeId, $employeeId, $projectId);
		$result->execute();
		$result->bind_result($stageId, $stageName);
		while($result->fetch())
		{
			array_push($stages, array("id" => $stageId, "name" => $stageName));
		}
		$result->close();
		while($db_connection->next_result()) $db_connection->store_result();

		for($idx=0;$idx<count($stages);$idx++)
		{
			$columnNames = array();
			$result = $db_connection->prepare("call fillReport_getStageColumns(?)");
			$result->bind_param('i', $stages[$idx]["id"]);
			$result->execute();
			$result->bind_result($columnName);
			while($result->fetch())
			{
				array_push($rows[1], 1);
				array_push($rows[1], $columnName);
				array_push($columnNames, $columnName);
			}
			$result->close();
			array_push($rows[0], count($columnNames));
			array_push($rows[0], $stages[$idx]["name"]);
			while($db_connection->next_result()) $db_connection->store_result();
		}

		for($idx=0;$idx<count($stages);$idx++)
		{
			$cnt = 2;
			$result = $db_connection->prepare("call fillReport_getStageReport(? ,?)");
			$result->bind_param('ii', $employeeId, $stages[$idx]["id"]);
			$result->execute();
			$meta = $result->result_metadata();
			while ($field = $meta->fetch_field()) 
			{ 
				$params[] = &$row[$field->name];
			}
			call_user_func_array(array($result, 'bind_result'), $params);
			
			while ($result->fetch()) 
			{
				if($idx==0)
				{
					$rows[$cnt] = array();
				}
				$rcnt = 0;
				foreach($row as $key => $val) 
				{
					if($idx==0 || $rcnt>=count($rowNames))
					{
						if($val!='')
							array_push($rows[$cnt],$val);
					}
					$rcnt++;
				}
				$cnt++;
			}
			$result->close();
			
			while($db_connection->next_result()) $db_connection->store_result();
		}

		$out .= '<table class="item">';
		$out .= '<tr>';
		$out .= '<th rowspan=2 colspan='.$rows[0][0].'>'.$rows[0][1].'</th>';
		for($i=2;$i<count($rows[0]);$i+=2)
		{
			$out .= '<th rowspan=1 colspan='.$rows[0][$i].'>'.$rows[0][$i+1].'</th>';
		}
		$out .= '</tr>';
		$out .= '<tr>';
		for($i=0;$i<count($rows[1]);$i+=2)
		{
			$out .= '<th colspan='.$rows[1][$i].'>'.$rows[1][$i+1].'</th>';
		}
		$out .= '</tr>';
		for($i=2;$i<count($rows);$i++)
		{
			$out .= '<tr>';
			for($j=0;$j<count($rows[$i]);$j++)
			{
				$out .= '<td>'.$rows[$i][$j].'</td>';
			}
			$out .= '</tr>';
		}
		$out .= '</table>';
		echo $out;
		
		break;

	case 'GET_FORMHEADER':
		$projectId = $_REQUEST['projectId'];
		$stageId = $_REQUEST['stageId'];
		
		$sql = "call data_getFormHeader (?, ?);";
		$result = $db_connection->prepare($sql);
		$result->bind_param('ii', $projectId, $stageId);
		$result->execute();
		
		$result->bind_result($id, $caption, $field, $editor, $type, $mandatory, $listId);

		$obj = array("columns" => array());
		$obj['fields'] = array();

		array_push($obj['columns'], array(header => 'id'));
		$obj['columns'][count($obj['columns'])-1]['hidden'] = true;
		$obj['columns'][count($obj['columns'])-1]['dataIndex'] = 'id';
		array_push($obj['fields'], array(name => 'id'));
		//$obj['fields'][count($obj['fields'])-1]['type'] = 'integer';
		
		
		array_push($obj['columns'], array(header => 'Бренд'));
		$obj['columns'][count($obj['columns'])-1]['dataIndex'] = 'name';
		array_push($obj['fields'], array(name => 'name'));
		//$obj['fields'][count($obj['fields'])-1]['type'] = 'string';
		
		while ($result->fetch()) {
			array_push($obj['fields'], array(name => $field));
			//$obj['fields'][count($obj['fields'])-1]['type'] = $type;
			
			array_push($obj['columns'], array(header => $caption));
			$obj['columns'][count($obj['columns'])-1]['dataIndex'] = $field;
			$obj['columns'][count($obj['columns'])-1]['width'] = 'auto';
			if ($editor == 1){
				$xtype = ($type == "integer" || $type == "double" || $type == "string" || $type == "numeric" ? "textfield" : $type);
				$obj['columns'][count($obj['columns'])-1]['editor'] = new StdClass();
				$obj['columns'][count($obj['columns'])-1]['editor']->xtype = $xtype;
				$obj['columns'][count($obj['columns'])-1]['editor']->allowBlank = !(bool)$mandatory;
				if ($xtype == "combo"){
					$obj['columns'][count($obj['columns'])-1]['editor']->listId = $listId;
				}
			}
		}
		
		$result->close();
		
		break;

	case 'GET_LIST':
		$listId = $_REQUEST['listId'];
		
		$result = $db_connection->prepare("call getList(?)");
		$result->bind_param('i', $listId);
		$result->execute();
		$result->bind_result($id, $name);
		
		$obj = array("records" => array());	
		
		while($result->fetch())
		{
			array_push($obj['records'], array(rowId => $id, name => $name));
		}
		$result->close();
		break;
}

if (isset($obj)){
	echo json_encode($obj);
}

$db_connection->close();
