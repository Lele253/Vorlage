<template>
  <v-container>
    <!-- Kartenreihe -->
    <v-row class="mb-4" dense>
      <v-col cols="12" sm="6" md="3" v-for="(value, key) in gewinnInfos" :key="key">
        <v-card>
          <v-card-title class="text-subtitle-1">{{ key }}</v-card-title>
          <v-card-text class="text-h6">{{ value.toFixed(2) }} €</v-card-text>
        </v-card>
      </v-col>
    </v-row>

    <!-- Buttons -->
    <v-btn  class="mb-4" color="primary" base-color="white" @click="openAddDialog">+ Bestand hinzufügen</v-btn>
    <v-btn class="mb-4 ml-2" color="success" base-color="white" @click="confirmAusschüttung = true">Gewinn ausschütten</v-btn>

    <!-- Desktop: Tabelle -->
    <v-card v-if="!isMobile" style="max-height: 650px; overflow-y: auto; position: relative">
      <div style="position: sticky; top: 0; z-index: 10; background: white">
        <v-text-field v-model="search" @click:clear="search = ''" clearable label="Suchen" class="mx-4" />
        <v-switch
            v-model="exakteSuche"
            label="Exakte Sorten-Suche aktivieren"
            :color="exakteSuche ? 'green' : ''"
            class="mx-4"
        />
        <v-alert type="info" border="start" variant="tonal" class="mx-4 mb-2" v-if="filteredVapes.length">
          <div><strong>Gefundene Vapes:</strong> {{ filteredVapes.length }}</div>
          <div><strong>Gesamteinkaufspreis:</strong> {{ einkaufSumme?.toFixed(2) || '0.00' }} €</div>
          <div><strong>Bereits verkauft:</strong> {{ verkauftAnzahl }}</div>
          <div><strong>Gesamtverkaufspreis:</strong> {{ verkaufSumme.toFixed(2) }} €</div>
          <div><strong>Gewinn (bereits verkauft):</strong> {{ gewinnSumme.toFixed(2) }} €</div>
        </v-alert>
      </div>


      <table class="v-table" style="width: 100%">
        <thead>
        <tr>
          <th @click="sortBy('id')">ID</th>
          <th @click="sortBy('sorte')">Sorte</th>
          <th @click="sortBy('einkaufspreisEinzeln')">Einkauf (€)</th>
          <th @click="sortBy('verkaufspreisEinzeln')">Verkauf (€)</th>
          <th @click="sortBy('verkaeufer')">Verkäufer</th>
          <th @click="sortBy('verkauft')">Verkauft</th>
        </tr>
        </thead>
        <tbody>
        <tr v-for="vape in filteredVapes" :key="vape.id">
          <td class="text-center">{{ vape.id }}</td>
          <td class="text-center">{{ vape.sorte }}</td>
          <td class="text-center">{{ vape.einkaufspreisEinzeln }}</td>
          <td class="text-center">{{ vape.verkaufspreisEinzeln || '-' }}</td>
          <td class="text-center">{{ vape.verkaeufer || '-' }}</td>
          <td class="text-center">{{ vape.verkauft ? 'Ja' : 'Nein' }}</td>
        </tr>
        </tbody>
      </table>
    </v-card>

    <!-- Mobile: Verkaufsformular -->
    <v-card v-else class="pa-4">
      <v-autocomplete
          v-model="mobileVerkauf.sorte"
          :items="groupedVapes"
          item-title="label"
          label="Vape auswählen"
          return-object
          :item-props="item => ({ title: `${item.sorte} (${item.count})` })"
      />
      <v-select
          v-model="mobileVerkauf.verkaeufer"
          :items="verkaeufer"
          label="Verkäufer"
      />
      <v-text-field
          v-model.number="mobileVerkauf.anzahl"
          type="number"
          label="Anzahl verkaufen"
          :max="mobileVerkauf.sorte?.count || 1"
          min="1"
      />
      <v-text-field
          v-model="mobileVerkauf.verkaufspreis"
          type="number"
          label="Verkaufspreis (€)"
      />
      <v-btn class="mt-2" color="primary" base-color="white" @click="verkaufen(mobileVerkauf.vape)">Verkaufen</v-btn>
    </v-card>

    <!-- Dialog: Verkauf bearbeiten -->
    <v-dialog v-model="dialog" max-width="400">
      <v-card>
        <v-card-title>Verkauf bearbeiten</v-card-title>
        <v-card-text>
          <div><strong>Sorte:</strong> {{ selectedVape?.sorte }}</div>
          <v-select
              v-model="selectedVape.verkaeufer"
              :items="verkaeufer"
              label="Verkäufer"
          />
          <v-text-field
              v-model="selectedVape.verkaufspreis"
              type="number"
              label="Verkaufspreis (€)"
          />
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn style="background-color: white !important;" color="primary" base-color="white" text @click="dialog = false">Abbrechen</v-btn>
          <v-btn base-color="white" color="green" style="background-color: white !important;" @click="verkaufen(selectedVape); dialog = false">Speichern</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Dialog: Neuer Bestand -->
    <v-dialog v-model="addDialog" max-width="500">
      <v-card>
        <v-card-title>Neuen Bestand hinzufügen</v-card-title>
        <v-card-text>
          <v-text-field v-model="newVape.sorte" label="Sorte" required />
          <v-text-field v-model="newVape.anzahl" label="Anzahl" type="number" required />
          <v-text-field v-model="newVape.einkaufspreis" label="Einkaufspreis (€)" type="number" required />
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn text color="primary" base-color="white" style="background-color: white !important;" @click="addDialog = false">Abbrechen</v-btn>
          <v-btn color="green" base-color="white" style="background-color: white !important;" @click="hinzufügen">Hinzufügen</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Dialog: Gewinn ausschütten bestätigen -->
    <v-dialog v-model="confirmAusschüttung" max-width="400">
      <v-card>
        <v-card-title>Gewinn ausschütten?</v-card-title>
        <v-card-text>Willst du den Gewinn jetzt ausschütten und aufteilen?</v-card-text>
        <v-card-item>
          <div>
            {{ gewinnInfos["Leandro (seit letzter Ausschüttung)"].toFixed(2) }} €<br>
            {{ gewinnInfos["Stefan (seit letzter Ausschüttung)"].toFixed(2) }} €<br>
            {{ gewinnInfos["Cosmin (seit letzter Ausschüttung)"].toFixed(2) }} €<br>
          </div>

        </v-card-item>
        <v-card-actions>
          <v-spacer />
          <v-btn color="primary" base-color="white" style="background-color: white !important;" text @click="confirmAusschüttung = false">Abbrechen</v-btn>
          <v-btn color="green" base-color="white" style="background-color: white !important;" @click="gewinnAusschuetten">Bestätigen</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Dialog: Gewinnausschüttung Ergebnis -->
    <v-dialog v-model="ausschüttungDialog" max-width="500">
      <v-card>
        <v-card-title>Gewinn wurde ausgeschüttet</v-card-title>
        <v-card-text>
          <div v-if="ausschüttungErgebnis">
            <p><strong>Leandro:</strong> {{ ausschüttungErgebnis.Leandro.toFixed(2) }} €</p>
            <p><strong>Stefan:</strong> {{ ausschüttungErgebnis.Stefan.toFixed(2) }} €</p>
            <p><strong>Cosmin:</strong> {{ ausschüttungErgebnis.Cosmin.toFixed(2) }} €</p>
          </div>
          <div v-else>
            <v-progress-circular indeterminate />
          </div>
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn color="primary" @click="ausschüttungDialog = false">Schließen</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>
</template>

<script>
export default {
  name: "VapeVerwaltung",
  data() {
    return {
      vapes: [],
      verkaeufer: ["LEANDRO", "STEFAN", "COSMIN"],
      search: "",
      isMobile: false,
      dialog: false,
      addDialog: false,
      confirmAusschüttung: false,
      ausschüttungDialog: false,
      ausschüttungErgebnis: null,
      selectedVape: null,
      sortKey: 'id',
      exakteSuche: false,
      sortDesc: false,
      newVape: {
        sorte: '',
        anzahl: 1,
        einkaufspreis: null,
      },
      mobileVerkauf: {
        sorte: null,
        verkaeufer: null,
        verkaufspreis: null,
        anzahl: 1,
      },
      gewinnInfos: {
        "Gesamtgewinn": 0,
        "Leandro (seit letzter Ausschüttung)": 0,
        "Stefan (seit letzter Ausschüttung)": 0,
        "Cosmin (seit letzter Ausschüttung)": 0,
      },
    };
  },
  computed: {
    gewinnSumme() {
      return this.filteredVapes
          .filter(v => v.verkauft)
          .reduce((sum, v) => sum + ((v.verkaufspreisEinzeln || 0) - (v.einkaufspreisEinzeln || 0)), 0);
    },
    verkauftSumme() {
      return this.filteredVapes
          .filter(v => v.verkauft)
          .reduce((sum, v) => sum + (v.verkaufspreisEinzeln || 0), 0);
    },
    verkaufSumme() {
      return this.filteredVapes.reduce((sum, v) => sum + (v.verkaufspreisEinzeln || 0), 0);
    },
    groupedVapes() {
      const grouped = {};
      this.vapes.forEach(v => {
        if (!v.verkauft) {
          grouped[v.sorte] = grouped[v.sorte] || [];
          grouped[v.sorte].push(v);
        }
      });

      return Object.entries(grouped).map(([sorte, vapes]) => ({
        sorte,
        count: vapes.length,
      }));
    },
    filteredVapes() {
      const query = this.search.toLowerCase().trim();

      return this.vapes
          .filter(v => {
            const allValues = Object.values(v).map(val => String(val).toLowerCase());

            if (this.exakteSuche) {
              return v.sorte?.toLowerCase() === query;
            } else {
              const terms = query.split(" ").filter(Boolean);
              return terms.every(term =>
                  allValues.some(val => val.includes(term))
              );
            }
          })
          .sort((a, b) => {
            const aVal = a[this.sortKey];
            const bVal = b[this.sortKey];
            if (aVal == null) return 1;
            if (bVal == null) return -1;
            return this.sortDesc
                ? (aVal < bVal ? 1 : -1)
                : (aVal > bVal ? 1 : -1);
          });
    },

    einkaufSumme() {
      return this.filteredVapes.reduce((sum, v) => sum + (v.einkaufspreisEinzeln || 0), 0);
    },
    verkauftAnzahl() {
      return this.filteredVapes.filter(v => v.verkauft).length;
    },
  },
  mounted() {
    this.checkViewport();
    this.loadDaten();
    window.addEventListener("resize", this.checkViewport);
  },
  unmounted() {
    window.removeEventListener("resize", this.checkViewport);
  },
  methods: {
    checkViewport() {
      this.isMobile = window.innerWidth < 768;
    },
    sortBy(key) {
      if (this.sortKey === key) {
        this.sortDesc = !this.sortDesc;
      } else {
        this.sortKey = key;
        this.sortDesc = false;
      }
    },
    async loadDaten() {
      try {
        const res = await fetch("https://franke-arts.de:8843/auth/vapes");
        const data = await res.json();
        this.vapes = data.map(v => ({
          ...v,
          verkaeufer: v.verkaeufer || null,
          verkaufspreis: v.verkaufspreis || null,
        }));

        const gewinnRes = await fetch("https://franke-arts.de:8843/auth/vapes/gewinn/verteilung");
        const gewinne = await gewinnRes.json();
        const gesamtRes = await fetch("https://franke-arts.de:8843/auth/vapes/gewinn/gesamt");
        const gesamt = await gesamtRes.json();

        this.gewinnInfos["Gesamtgewinn"] = gesamt;
        this.gewinnInfos["Leandro (seit letzter Ausschüttung)"] = gewinne.Leandro || 0;
        this.gewinnInfos["Stefan (seit letzter Ausschüttung)"] = gewinne.Stefan || 0;
        this.gewinnInfos["Cosmin (seit letzter Ausschüttung)"] = gewinne.Cosmin || 0;
      } catch (e) {
        console.error("Fehler beim Laden", e);
      }
    },
    async verkaufen() {
      const isMobile = this.isMobile;

      if (isMobile) {
        const sorte = this.mobileVerkauf.sorte?.sorte;
        const { verkaeufer, verkaufspreis, anzahl } = this.mobileVerkauf;

        if (!sorte) return alert("Bitte Sorte auswählen.");
        if (!verkaeufer || isNaN(parseFloat(verkaufspreis))) {
          return alert("Bitte Verkäufer und Preis korrekt angeben.");
        }
        if (!anzahl || isNaN(anzahl) || anzahl <= 0) {
          return alert("Bitte gültige Anzahl angeben.");
        }

        const freieVapes = this.vapes.filter(v => !v.verkauft && v.sorte === sorte);

        if (freieVapes.length < anzahl) {
          return alert(`Nur ${freieVapes.length} freie Geräte verfügbar für ${sorte}.`);
        }

        const zuVerkaufen = freieVapes.slice(0, anzahl);

        for (const vape of zuVerkaufen) {
          await fetch(`https://franke-arts.de:8843/auth/vapes/${vape.id}/verkauft`, {
            method: "PUT",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
              verkaeufer,
              verkaufspreis: parseFloat(verkaufspreis),
            }),
          });
        }

        this.mobileVerkauf = { sorte: null, verkaeufer: null, verkaufspreis: null, anzahl: 1 };
        this.loadDaten();
        alert(`${anzahl}x ${sorte} erfolgreich verkauft.`);
      }

      // Desktop-Logik bleibt unverändert
    },
    // ... (restliche Methoden bleiben gleich)
    openDialog(vape) {
      this.selectedVape = { ...vape };
      this.dialog = true;
    },
    openAddDialog() {
      this.newVape = { sorte: '', anzahl: 1, einkaufspreis: null };
      this.addDialog = true;
    },
    async hinzufügen() {
      const { sorte, anzahl, einkaufspreis } = this.newVape;
      if (!sorte || !anzahl || !einkaufspreis) return alert("Bitte alle Pflichtfelder ausfüllen.");

      try {
        await fetch("https://franke-arts.de:8843/auth/vapes/neuer-bestand", {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ sorte, anzahl: parseInt(anzahl), einkaufspreis: parseFloat(einkaufspreis) }),
        });
        this.addDialog = false;
        await this.loadDaten();
      } catch (e) {
        console.error("Fehler beim Hinzufügen", e);
      }
    },
    async gewinnAusschuetten() {
      this.confirmAusschüttung = false;
      this.ausschüttungErgebnis = null;
      this.ausschüttungDialog = true;

      try {
        const res = await fetch("https://franke-arts.de:8843/auth/vapes/gewinn/ausschuetten", {
          method: "POST"
        });
        const data = await res.json();
        this.ausschüttungErgebnis = data;
        await this.loadDaten(); // Gewinne neu laden
      } catch (e) {
        console.error("Fehler bei Ausschüttung", e);
        this.ausschüttungDialog = false;
      }
    }
    ,
  },
};
</script>

<style scoped>
th {
  cursor: pointer;
}

v-btn {
  background-color: white !important;
  color: white !important;
}
</style>
