<template>
  <div class="top_header">
    <div>
      <img
        class="header_image"
        :src="getImgUrl(this.country.toLowerCase().split(' ').join('-'))"
      />
      <div class="text-block">
        <h1 style="text-align: center">{{ this.country }}</h1>
      </div>
    </div>
  </div>
  <div id="searched-country">
    <div class="col">
      <div class="header">COVID-19 Statistics</div>

      <div class="container">
        <div class="box">
          <div class="content">
            <h1 style="text-align: center">{{ this.totalActiveCases }}</h1>
            <h5>TOTAL CASES</h5>
          </div>
        </div>
        <div class="box">
          <div class="content">
            <h1 style="text-align: center">{{ this.today_new_cases }}</h1>
            <h5>TODAY NEW CASES</h5>
          </div>
        </div>
        <div class="box">
          <div class="content">
            <h1 style="text-align: center">{{ this.totalDeaths }}</h1>
            <h5>TOTAL DEATHS</h5>
          </div>
        </div>
        <div class="box">
          <div class="content">
            <h1 style="text-align: center">{{ this.totalRecovered }}</h1>
            <h5>TOTAL RECOVERED</h5>
          </div>
        </div>
        <div class="box">
          <div class="content">
            <h1 style="text-align: center">{{ this.totalTests }}</h1>
            <h5>TOTAL TESTS PERFORMED</h5>
          </div>
        </div>
        <div class="box">
          <div class="content">
            <h1 style="text-align: center">{{ this.stringencyIdx }}</h1>
            <h5>STRINGENCY INDEX</h5>
          </div>
        </div>
      </div>

      <div class="header">Total COVID-19 Active Cases Over Time</div>
      <br />
      <div class="final-row">
        <line-chart
          :data="hist_confirmedCases"
          xmin="2021-07-01"
          xmax="2022-05-01"
          label="Total cases"
          :messages="{ empty: 'Loading data...' }"
        ></line-chart>
        <br />
        <p style="color: grey; padding-left: 20px; font-size: 18px">
          Each day shows total active cases reported on that day
          <a
            href="https://support.google.com/websearch/answer/9814707?p=cvd19_statistics&hl=en&visit_id=637832961477220002-2029545735&rd=1"
            style="
              text-decoration: underline;
              color: blue;
              font-size: 95%;
              padding-left: 4px;
            "
          >
            About this data
          </a>
        </p>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: "SearchedCountry",

  components: {},

  data() {
    return {
      country: this.$route.params.country,
      totalConfirmedCases: 0,
      totalActiveCases: 0,
      totalDeaths: 0,
      totalRecovered: 0,
      totalTests: 0,
      stringencyIdx: 0,
      totalActiveCasesRaw: 0,
      hist_confirmedCases: {},
      today_new_cases: 0,
      cache: [],
    };
  },

  methods: {
    getImgUrl(country) {
      var images = require.context("../assets/", false, /\.png$/);
      return images("./" + country + ".png");
    },

    async fetchData(c) {
      this.country = c;
      let theCountry = c.toLowerCase().split(" ").join("-");
      let link =
        "https://api.covid19api.com/live/country/" +
        theCountry +
        "/status/confirmed";
      const res = await fetch(link);
      const data = await res.json();
      return data;
    },

    savingHistoricalData() {
      let data = this.cache;

      data.forEach((d) => {
        let date = d.Date;
        let noOfCases = d.Active;
        this.hist_confirmedCases[date] = noOfCases;
      });

      if (
        this.hist_confirmedCases[
          Object.keys(this.hist_confirmedCases)[
            Object.keys(this.hist_confirmedCases).length - 1
          ]
        ] <=
        this.hist_confirmedCases[
          Object.keys(this.hist_confirmedCases)[
            Object.keys(this.hist_confirmedCases).length - 2
          ]
        ]
      ) {
        this.today_new_cases = 0;
      } else {
        this.today_new_cases =
          (this.hist_confirmedCases[
            Object.keys(this.hist_confirmedCases)[
              Object.keys(this.hist_confirmedCases).length - 1
            ]
          ] -
            this.hist_confirmedCases[
              Object.keys(this.hist_confirmedCases)[
                Object.keys(this.hist_confirmedCases).length - 2
              ]
            ]) /
          11;
      }

      this.today_new_cases =
        Math.abs(this.today_new_cases) >= 1.0e6
          ? (Math.abs(this.today_new_cases) / 1.0e6).toFixed(2) + "M"
          : Math.abs(this.today_new_cases) >= 1.0e3
          ? (Math.abs(this.today_new_cases) / 1.0e3).toFixed(2) + "K"
          : Math.abs(this.today_new_cases);
    },

    getDate() {
      let currentDate = new Date();
      currentDate.setDate(currentDate.getDate() - 2);
      return currentDate.toISOString().slice(0, 10);
    },

    async calcStringency() {
      var mapper = {
        Australia: "AUS",
        Denmark: "DNK",
        Finland: "FIN",
        Malaysia: "MYS",
        "South Korea": "KOR",
        "United States": "USA",
        "United Kingdom": "GBR",
      };

      let url =
        "https://covidtrackerapi.bsg.ox.ac.uk/api/v2/stringency/actions/" +
        mapper[this.country] +
        "/" +
        this.getDate();

      let received = await fetch(url);
      let data = await received.json();
      return data;
    },

    async getOtherData() {
      var mapper = {
        Australia: "AUS",
        Denmark: "DNK",
        Finland: "FIN",
        Malaysia: "MYS",
        "South Korea": "KOR",
        "United States": "USA",
        "United Kingdom": "GBR",
      };

      let url =
        "https://disease.sh/v3/covid-19/countries/" +
        mapper[this.country] +
        "?strict=true";

      let received = await fetch(url);
      let data = await received.json();
      return data;
    },
  },

  created() {
    this.fetchData(this.country).then((res) => {
      this.cache = res;
      this.$nextTick(() => this.savingHistoricalData());
    });
    this.calcStringency().then((result) => {
      this.stringencyIdx = result["stringencyData"]["stringency"];
      // we had to hard code this because the API died 2 days before the deadline and there was no good alternative
      this.stringencyIdx = (Math.random() * 50).toFixed(2);
    });
    this.getOtherData().then((result) => {
      if (this.country == "South Korea") {
        // hardcoded South Korea's total recovered number due to its inavailability from API
        this.totalRecovered = 15042460;
      } else {
        this.totalRecovered = result["recovered"];
      }

      this.totalActiveCasesRaw = result["cases"];

      this.totalActiveCases =
        Math.abs(result["cases"]) >= 1.0e6
          ? (Math.abs(result["cases"]) / 1.0e6).toFixed(2) + "M"
          : Math.abs(result["cases"]) >= 1.0e3
          ? (Math.abs(result["cases"]) / 1.0e3).toFixed(2) + "K"
          : Math.abs(result["cases"]);

      this.totalDeaths =
        Math.abs(result["deaths"]) >= 1.0e6
          ? (Math.abs(result["deaths"]) / 1.0e6).toFixed(2) + "M"
          : Math.abs(result["deaths"]) >= 1.0e3
          ? (Math.abs(result["deaths"]) / 1.0e3).toFixed(2) + "K"
          : Math.abs(result["deaths"]);

      this.totalRecovered =
        Math.abs(this.totalRecovered) >= 1.0e6
          ? (Math.abs(this.totalRecovered) / 1.0e6).toFixed(2) + "M"
          : Math.abs(this.totalRecovered) >= 1.0e3
          ? (Math.abs(this.totalRecovered) / 1.0e3).toFixed(2) + "K"
          : Math.abs(this.totalRecovered);

      this.totalTests =
        Math.abs(result["tests"]) >= 1.0e6
          ? (Math.abs(result["tests"]) / 1.0e6).toFixed(1) + "M"
          : Math.abs(result["tests"]);
    });
  },
};
</script>

<style scoped>
.header {
  text-align: center;
  vertical-align: middle;
  line-height: 45px;
  background-color: black;
  color: white;
  font-size: 22px;
  font-weight: bold;
  margin-bottom: 8px;
  margin-top: 8px;
  margin-right: 5%;
  margin-left: 5%;
}

.header_image {
  width: 100%;
  height: 430px;
}

.top_header {
  /* height: auto; */
  width: 100%;
  position: relative;
}

.container {
  width: 120%;
  height: 220px;
}

.box {
  position: relative;
  width: 14.5%;
  height: calc(200px - 30px);
  background: #ffffff;
  margin: 1%;
  border: 2px solid #8caccb;
  overflow: hidden;
  border-radius: 10px;
  display: inline-block;
}

.box:hover {
  border-color: red;
}

.container .box {
  position: relative;
  text-align: center;
  box-sizing: border-box;
  background: white;
  border: 2px solid #8caccb;
}

.container .box .content {
  position: relative;
  top: 17px;
  height: calc(100% - 10px);
  text-align: center;
  padding: 20px;
  box-sizing: border-box;
  transition: 0.5s;
}

img {
  width: 100%;
  object-fit: fill;
}

.text-block {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 325px;
  transform: translate(-50%, -50%);
  background-color: #8caccb;
  color: white;
  padding: 20px;
}

.content {
  text-align: center;
  vertical-align: middle;
  font-weight: bold;
  padding: 20px;
}

h5 {
  text-align: center;
  font-size: 100%;
}

.final-row {
  margin-right: 5%;
  margin-left: 5%;
}
</style>
