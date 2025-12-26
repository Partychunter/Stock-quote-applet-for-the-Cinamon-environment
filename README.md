# Stock-quote-applet-for-the-Cinamon-environment
Stock exchange rates applet for the Cinnamon environment - displays exchange rates and several selected indices relevant to a Polish resident on the Panel and in an expandable list

## Installation and use
Version 1: 
you need to create a directory "gielda@mojaplet" in .local/share/cinnamon/applets/gielda@mojaplet

>>/home/user/.local/share/cinnamon/applets/gielda@mojaplet
>>
We place 3 files in the folder: applet.js, metadata.json, stylesheet.css


metadata.json
```{
    "uuid": "gielda@mojaplet",
    "name": "Giełda - Notowania",
    "description": "Notowania giełdowe (Yahoo + Stooq)",
    "version": "1.0",
    "icon": "emblem-money"
}
```

stylesheet.css
```
.popup-header {
    font-weight: bold;
    font-size: 1.1em;
    padding: 5px 10px;
    color: #f8fafc;
    background-color: rgba(255, 255, 255, 0.05);
}

.stock-row {
    padding: 6px 10px;
    spacing: 5px; /* Odstęp między elementami */
}

/* Kolumny - sztywne szerokości dla wyrównania */
.stock-name-box {
    width: 110px;
}

.stock-price {
    width: 80px;
    text-align: right;
    font-family: monospace;
    font-weight: bold;
    color: #f8fafc;
}

.stock-currency {
    width: 40px;
    text-align: center;
    color: #94a3b8;
    font-size: 0.85em;
    border: 1px solid #475569;
    border-radius: 4px;
    padding: 1px;
}

.stock-change {
    width: 75px;
    text-align: right;
    font-weight: bold;
}

.stock-graph {
    width: 60px; 
    height: 25px;
}

.stock-period {
    width: 30px;
    text-align: right;
    font-size: 0.8em;
    color: #64748b;
}

/* Kolory */
.change-up { color: #10b981; }
.change-down { color: #ef4444; }
.text-sub { font-size: 0.85em; color: #94a3b8; font-weight: normal; }

```

applet.js
```
const Applet = imports.ui.applet;
const St = imports.gi.St;
const Main = imports.ui.main;
const PopupMenu = imports.ui.popupMenu;
const Lang = imports.lang;
const Mainloop = imports.mainloop;
const GLib = imports.gi.GLib;
const Gio = imports.gi.Gio;
const Cairo = imports.cairo; // Potrzebne do rysowania wykresów

// --- KONFIGURACJA ---
const TICKERS = [
    { provider: 'yahoo', symbol: 'EURPLN=X',  name: 'EUR/PLN', sub: 'Euro Złoty',  curr: 'PLN', flag: '🇪🇺' },
    { provider: 'yahoo', symbol: 'USDPLN=X',  name: 'USD/PLN', sub: 'Dolar Złoty', curr: 'PLN', flag: '🇺🇸' },
    { provider: 'yahoo', symbol: 'EURUSD=X',  name: 'EUR/USD', sub: 'Euro Dolar',  curr: 'USD', flag: '🇺🇸' },
    { provider: 'yahoo', symbol: '^NDX',      name: 'Nasdaq',  sub: 'USA Tech',    curr: 'USD', flag: '💻' },
    
    // Stooq dla polskich (mWIG40, sWIG80)
    { provider: 'stooq', symbol: 'mwig40',    name: 'mWIG40',  sub: 'Warszawa',    curr: 'PLN', flag: '🇵🇱' },
    { provider: 'stooq', symbol: 'swig80',    name: 'sWIG80',  sub: 'Warszawa',    curr: 'PLN', flag: '🇵🇱' },
    
    { provider: 'yahoo', symbol: '^GDAXI',    name: 'DAX',     sub: 'Niemcy',      curr: 'EUR', flag: '🇩🇪' },
    { provider: 'yahoo', symbol: '^N225',     name: 'Nikkei',  sub: 'Japonia',     curr: 'JPY', flag: '🇯🇵' },
    { provider: 'yahoo', symbol: '^FCHI',     name: 'CAC 40',  sub: 'Francja',     curr: 'EUR', flag: '🇫🇷' },
    { provider: 'yahoo', symbol: '^FTSE',     name: 'FTSE',    sub: 'Londyn',      curr: 'GBP', flag: '🇬🇧' }
];

function MyApplet(metadata, orientation, panel_height, instance_id) {
    this._init(metadata, orientation, panel_height, instance_id);
}

MyApplet.prototype = {
    __proto__: Applet.TextIconApplet.prototype,

    _init: function(metadata, orientation, panel_height, instance_id) {
        try {
            Applet.TextIconApplet.prototype._init.call(this, orientation, panel_height, instance_id);
            this.setAllowedLayout(Applet.AllowedLayout.BOTH);
            
            this.stockData = [];
            this.currentIndex = 0;
            
            // Ukrywamy ikonę systemową, bo mamy flagi
            this.hide_applet_icon();
            this.set_applet_label("Start...");

            // Menu
            this.menuManager = new PopupMenu.PopupMenuManager(this);
            this.menu = new Applet.AppletPopupMenu(this, orientation);
            this.menuManager.addMenu(this.menu);

            // Nagłówek
            let headerItem = new PopupMenu.PopupMenuItem("Notowania", { reactive: false });
            headerItem.actor.add_style_class_name("popup-header");
            this.menu.addMenuItem(headerItem);

            // Lista przewijana
            this.scrollSection = new PopupMenu.PopupMenuSection();
            this.menu.addMenuItem(this.scrollSection);

            // Pobieranie
            this._refreshData();
            this._dataTimer = Mainloop.timeout_add_seconds(300, Lang.bind(this, this._refreshData));
            this._panelTimer = Mainloop.timeout_add_seconds(10, Lang.bind(this, this._rotatePanel));

        } catch (e) {
            global.logError("Init Error: " + e);
            this.set_applet_label("ERR");
        }
    },

    on_applet_clicked: function(event) {
        this.menu.toggle();
    },

    on_applet_removed_from_panel: function() {
        if (this._dataTimer) Mainloop.source_remove(this._dataTimer);
        if (this._panelTimer) Mainloop.source_remove(this._panelTimer);
    },

    _refreshData: function() {
        if (this.stockData.length === 0) {
            this.stockData = new Array(TICKERS.length).fill(null);
        }

        TICKERS.forEach((item, index) => {
            let url = "";
            // Pobieramy trochę więcej danych (7 dni) żeby wykres był ładny
            if (item.provider === 'stooq') {
                url = `https://stooq.pl/q/d/l/?s=${item.symbol}&i=d`;
            } else {
                url = `https://query1.finance.yahoo.com/v8/finance/chart/${item.symbol}?interval=1d&range=7d`;
            }
            
            this._execCurl(url, (output) => {
                if (!output) return;
                
                if (item.provider === 'stooq') {
                    this._parseStooq(output, item, index);
                } else {
                    this._parseYahoo(output, item, index);
                }
                this._updateMenuIfReady();
            });
        });
        return true;
    },

    _execCurl: function(url, callback) {
        try {
            let argv = ['/usr/bin/curl', '-s', '-L', '-A', 'Mozilla/5.0', url];
            let [res, pid, in_fd, out_fd, err_fd] = GLib.spawn_async_with_pipes(
                null, argv, null, GLib.SpawnFlags.SEARCH_PATH | GLib.SpawnFlags.DO_NOT_REAP_CHILD, null
            );

            let out_reader = new Gio.DataInputStream({ base_stream: new Gio.UnixInputStream({fd: out_fd}) });
            
            let buffer = "";
            this._readStream(out_reader, (data) => {
                callback(data);
                GLib.spawn_close_pid(pid);
            });
        } catch (e) { callback(null); }
    },

    _readStream: function(stream, callback, currentData = "") {
        stream.read_line_async(0, null, (obj, res) => {
            try {
                let [line, len] = obj.read_line_finish(res);
                if (line !== null) {
                    let str = new TextDecoder("utf-8").decode(line);
                    this._readStream(stream, callback, currentData + str + "\n");
                } else {
                    callback(currentData);
                }
            } catch (e) { callback(currentData); }
        });
    },

    // --- PARSOWANIE (z historią) ---
    _parseYahoo: function(jsonString, item, index) {
        try {
            let json = JSON.parse(jsonString);
            if (!json.chart || !json.chart.result) return;
            
            let quotes = json.chart.result[0].indicators.quote[0].close;
            let clean = quotes.filter(q => q != null);
            
            this._processData(clean, item, index);
        } catch (e) {}
    },

    _parseStooq: function(csvString, item, index) {
        try {
            let lines = csvString.trim().split('\n');
            // Pomijamy nagłówek i bierzemy ostatnie linie
            // Stooq: Data,Open,High,Low,Close... (indeks 4 to Close)
            let history = [];
            // Bierzemy max 7 ostatnich dni
            let start = Math.max(1, lines.length - 7); 
            
            for (let i = start; i < lines.length; i++) {
                let parts = lines[i].split(',');
                let val = parseFloat(parts[4]);
                if (!isNaN(val)) history.push(val);
            }
            this._processData(history, item, index);
        } catch (e) {}
    },

    _processData: function(history, item, index) {
        if (history.length < 2) return;

        let current = history[history.length - 1];
        let prev = history[history.length - 2];
        let change = ((current - prev) / prev) * 100;

        // Jeśli mamy więcej niż 5 punktów, to przycinamy do wykresu (np. ostatnie 7)
        let chartData = history.slice(-7);

        this.stockData[index] = {
            item: item,
            price: current,
            change: change,
            history: chartData
        };
    },

    // --- RYSOWANIE WYKRESU (CAIRO) ---
    _createGraphActor: function(dataPoints, isUp) {
        let drawingArea = new St.DrawingArea({ style_class: 'stock-graph' });
        
        drawingArea.connect('repaint', Lang.bind(this, function(area) {
            let cr = area.get_context();
            let [w, h] = area.get_surface_size();
            
            // Konfiguracja linii
            cr.setLineWidth(1.5);
            // Kolor (zielony lub czerwony)
            if (isUp) cr.setSourceRGB(0.06, 0.73, 0.5); // #10b981
            else cr.setSourceRGB(0.94, 0.27, 0.27);     // #ef4444

            // Normalizacja danych do wymiarów obszaru
            let min = Math.min(...dataPoints);
            let max = Math.max(...dataPoints);
            let range = max - min || 1; // unikamy dzielenia przez 0
            
            // Marginesy w pionie (żeby wykres nie dotykał krawędzi)
            let padding = 2;
            let drawHeight = h - (padding * 2);

            for (let i = 0; i < dataPoints.length; i++) {
                let x = (i / (dataPoints.length - 1)) * w;
                // Odwracamy Y (0 jest na górze)
                let y = h - padding - ((dataPoints[i] - min) / range) * drawHeight;
                
                if (i === 0) cr.moveTo(x, y);
                else cr.lineTo(x, y);
            }
            cr.stroke();
            cr.$dispose();
        }));
        
        return drawingArea;
    },

    // --- BUDOWANIE MENU ---
    _updateMenuIfReady: function() {
        this.scrollSection.removeAll();

        this.stockData.forEach(data => {
            if (!data) return;

            let row = new PopupMenu.PopupBaseMenuItem({ reactive: false });
            
            // Główny kontener wiersza
            let box = new St.BoxLayout({ style_class: 'stock-row' });

            // 1. Nazwa i Flaga + Sub (pionowo)
            let nameBox = new St.BoxLayout({ vertical: true, style_class: 'stock-name-box' });
            let labelName = new St.Label({ text: `${data.item.flag}  ${data.item.name}` });
            let labelSub = new St.Label({ text: data.item.sub, style_class: 'text-sub' });
            nameBox.add(labelName);
            nameBox.add(labelSub);

            // 2. Cena
            let labelPrice = new St.Label({ 
                text: data.price.toFixed(2), 
                style_class: 'stock-price' 
            });

            // 3. Waluta (Ramka)
            let labelCurr = new St.Label({ 
                text: data.item.curr, 
                style_class: 'stock-currency' 
            });

            // 4. Zmiana %
            let isUp = data.change >= 0;
            let sign = isUp ? "+" : "";
            let colorClass = isUp ? 'change-up' : 'change-down';
            let labelChange = new St.Label({ 
                text: `${sign}${data.change.toFixed(2)}%`, 
                style_class: `stock-change ${colorClass}` 
            });

            // 5. Wykres (Canvas/Cairo)
            let graphActor = this._createGraphActor(data.history, isUp);

            // 6. Okres
            let periodText = data.history.length >= 6 ? "7D" : "5D";
            let labelPeriod = new St.Label({ 
                text: periodText, 
                style_class: 'stock-period' 
            });

            // Składanie wiersza (kolejność jest ważna!)
            // Używamy kontenerów pośrednich do wyrównania w pionie (y_align)
            
            box.add(nameBox, { y_align: St.Align.MIDDLE, y_fill: false });
            box.add(labelPrice, { y_align: St.Align.MIDDLE });
            box.add(labelCurr, { y_align: St.Align.MIDDLE });
            box.add(labelChange, { y_align: St.Align.MIDDLE });
            box.add(graphActor, { y_align: St.Align.MIDDLE });
            box.add(labelPeriod, { y_align: St.Align.MIDDLE });

            row.addActor(box);
            this.scrollSection.addMenuItem(row);
        });
    },

    _rotatePanel: function() {
        let loaded = this.stockData.filter(d => d !== null);
        
        if (loaded.length === 0) {
            this.hide_applet_icon();
            this.set_applet_label("..."); 
            return true;
        }

        this.currentIndex = (this.currentIndex + 1) % loaded.length;
        let d = loaded[this.currentIndex];
        let isUp = d.change >= 0;
        let arrow = isUp ? "▲" : "▼";
        
        this.hide_applet_icon();
        this.set_applet_label(`${d.item.flag} ${d.item.name}: ${d.price.toFixed(2)} ${arrow}`);
        this.set_applet_tooltip(`${d.item.name}\n${d.change.toFixed(2)}%`);

        return true;
    }
};

function main(metadata, orientation, panel_height, instance_id) {
    return new MyApplet(metadata, orientation, panel_height, instance_id);
}

```
Once added, simply restart Cinamon, search for and add the applet Giełda - Notowania
