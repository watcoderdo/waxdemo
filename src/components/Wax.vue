<template>
  <div class="login-block">
    <div class="buttons">
      <button @click="login">Login</button>
      <button @click="autoharvest">AutoHarvest</button>
    </div>
    <div class="account-info" v-if="userAccount != ''">
      <p>{{ userName }}</p>
      <p>Active: {{ pubKeys[0] }}</p>
      <p>Owner: {{ pubKeys[1] }}</p>
    </div>
    <div class="login-status">
      <p><pre>Status: {{ loginStatus }}</pre></p>
      <p><pre>Actions : {{ actionStatus }} </pre></p>
      <p><pre>Debug : {{ debugStatus }} </pre></p>
    </div>
  </div>
</template>

<script>
/*
      ##########################   AUTO HARVEST FUNCTION ####################################

      Before using, please read all this section, but also settings help, and changelogs !

      This script is a fully harvesting feature you can run from your browser. It does the 
      folowing tasks, and can avoid harvest all tax/fee in the game :

      1. Collect last transactions for your account on the WAX block chain
      2. Retrieve last harvest datetime for assets you own (animals and buildings only for now)
      3. Harvest assets if they are ready
      4. Refill food /water if needed
      5. Plan the next harvesting cycle depending on the cycle (immediately, random, optimized)

      It can run in parallel with any other harvesting method (online gaming, TACO, ...)
      as it will adapt to real data from the blockchain. You can still get harvesting errors
      if some events are done in the same timing from another source

      Steps to use :
      1. Fork the projet if not done yet
      2. Setup your assets ID (see "USer Settings" section below), take your time and double check everything is correct
      3. Setup the blockchain history API call (should be fine with defaults)
      4. Setup the maximum hours of blockchain transaction history needed (no need for a long timing, 
      just setup this parameter as the longest harvest time for your asset, then round to upper integer value)
      5. If needed, adjust "std_harvest*" timings (it's working fine with defaults)
      6. Allow popup windows on your browser for the domain/link you are using
      7. Check the "Automic login" option on your wallet for login / harvest / refill food
      8. Make sure you have enough resources stacked (CPU/RAM/NET) for the calls not being rejected


      ###################### IMPORTANT NOTES AND INFORMATION ################################

      This code has been tested on my own account, but is provided as is. Use it at your own risk
      I know there could be a LOT of possible optimizations : loops / processing / vars / ...

      The main idea is to keep it simple, readable, and editable by anybody :
      - VAR names are pretty long and explicit, to keep it simple and very understable
      - Loops and conditions are kept in the most simple way (no complex or nested conditions)

      Any question ? Contact me : ctft (at) runbox.com
      Donations are appreciated but not required : 5ayzc.wam

*/

import * as waxjs from "@waxio/waxjs/dist";

export default {
  name: "App",
  data() {
    return {
      userAccount: "",
      pubKeys: "",
      wax: new waxjs.WaxJS("https://wax.greymass.com", null, null, false),
      result: "",
      userName: "",
      loginStatus: "User is not logged in",
      actionStatus: "N/A",
      debugStatus: "N/A",
    };
  },
  methods: {
    async login() {
      try {
        console.log("logging in through WCW");
        this.userAccount = await this.wax.login();
        this.pubKeys = this.wax.pubKeys;
        this.userName = this.wax.userAccount;
        this.loginStatus = `User has approved dApp access`;
      } catch (e) {
        console.log(e);
        this.loginStatus = `User denied dApp access`;
      }
    },
    async autoharvest() {
/*
      #####################################################################################
      ###################### USER SETTINGS - START SECTION ################################
      #####################################################################################

      First step is setting assets array as assetID (string !), Harvest Time (in minutes), AnimalType (for help reference only)
      [IMPROVE] : could have a dedicated table for HarvestTime, and even retrieve the assets direcly on the blockchain (easy as they are harvested regularly)

      Below is a memo for Animals, just copy/paste the line, then replace with you assetID
      Please note that assets shoud be stacked in the game first

      //ANIMALS

      ["assetid", 120, "animal-goose-common"],
      ["assetid", 260, "animal-goose-rare"],
      ["assetid", 400, "animal-goose-epic"],

      ["assetid", 60, "animal-cow-common"],
      ["assetid", 130, "animal-cow-rare"],
      ["assetid", 200, "animal-cow-epic"],

      ["assetid", 180, "animal-pig-common"],
      ["assetid", 390, "animal-pig-rare"],
      ["assetid", 600, "animal-pig-epic"],

      ["assetid", 120, "animal-goat-common"],
      ["assetid", 260, "animal-goat-rare"],
      ["assetid", 400, "animal-goat-epic"],

      ["assetid", 180, "animal-sheep-common"],
      ["assetid", 390, "animal-sheep-rare"],
      ["assetid", 600, "animal-sheep-epic"],

      ["assetid", 60, "animal-hen-common"],
      ["assetid", 130, "animal-hen-rare"],
      ["assetid", 200, "animal-hen-epic"],

      //BUILDINGs
      ["assetid", 60, "building-lumberjack"],
      ["assetid", 60, "building-fishingpier"],

      You can also double check your ASSETS on link https://wax.atomichub.io/explorer/asset/{ASSETID} 
      (of course please replace the {ASSETID} with the number)

*/
      let assets = [
        ["1099570058717", 130, "animal-cow-rare"],
        ["1099570059410", 130, "animal-cow-rare"],
        ["1099570059411", 130, "animal-cow-rare"],
        ["1099570058580", 120, "animal-goat-common"],
        ["1099570060519", 400, "animal-goat-epic"],
        ["1099570060521", 260, "animal-goat-rare"],
        ["1099570060518", 400, "animal-goose-epic"],
        ["1099570061147", 400, "animal-goose-epic"],
        ["1099570058713", 120, "animal-goose-common"],
        ["1099570058715", 120, "animal-goose-common"],
        ["1099570059409", 120, "animal-goose-common"],
        ["1099578017980", 120, "animal-goose-common"],
        ["1099578018922", 120, "animal-goose-common"],
        ["1099578188421", 120, "animal-goose-common"],
        ["1099578307316", 120, "animal-goose-common"],
        ["1099570058578", 390, "animal-pig-rare"],
        ["1099570060520", 390, "animal-pig-rare"],
        ["1099571446256", 600, "animal-pig-epic"],
        ["1099570058714", 180, "animal-pig-common"],
        ["1099570058577", 180, "animal-pig-common"],
        ["1099570058581", 180, "animal-pig-common"],
        ["1099570057821", 180, "animal-sheep-common"],
        ["1099585174662", 60, "building-lumberjack"],
        ["1099590552076", 60, "building-fishingpier"],
      ];

      //We use min and max value for checking harvesting timing delay
      //Timing can be quite low because this new function retrieves last transactions on the Blockchain and does not call any transaction if not necessary
      const std_harvest_min_wait_time_sec = 4 * 60; //MIN possible harvest check delay in seconds
      const std_harvest_max_wait_time_sec = 8 * 60; //MAX possible harvest check delay in seconds

      //Set how many hours of TX should we grab. This value should be setup according to the maximum Harvest time asset
      const required_hours_tx_history = 11;

      //How many transaction we should retrieve at once, then analyze looking for harvestanim action
      //Warning : this value depends on the node, and maximum value on Greymass is 100. It can be split in smaller numbers without any issue
      const nb_tx_tofetch_per_apicall = 100;

      //Should we use the Harvest ALL (hvstmultanim / hvstmultbuild / hvstmultgrdn) function included in the game / TACO
      //Be careful with enabling this setting as there are some tax/fee to use it
      const use_multi_harvest = false; //default is FALSE !


      //###################################################################################
      //###################### USER SETTINGS - END SECTION ################################
      //###################################################################################


      if (!this.wax.api) {
        return console.log("Need to Login first");
      }

      this.actionStatus = "Started";

      const delay = (ms) => new Promise((res) => setTimeout(res, ms));
      for (;;) {
        //MAIN LOOP FOR HARVESTING 
        this.actionStatus = this.actionStatus.concat("\n\nNew cycle started on ", Date(), " : " );

        let current_datetime = new Date();
        let found_something_toharvest = false; //We use this global setting to track if harvested at least once in this cycle. Useful for optimizing Next Harvest Time
        let harvested_something_withsuccess = false; //We use this global setting to track if at least 1 asset was harvested with success. Uselfil for optimizing next Harvest Time
        let optim_harvest_wait_time_sec = 3600; //Default value if not overwritten during auto-harvest cycles

        let multi_harvest_animal_ids = []; //List of animal asset_ids ready to be harvested, to be used with multi-harvest
        let multi_harvest_building_ids = []; //List of building asset_ids ready to be harvested, to be used with multi-harvest

        //Grabbing last blockchain transations, for at leat min_hours_tx_history HOURS
        let tx_nbhours = 0;
        let cpos = -1; //cpos is used to API, check https://developers.eos.io/manuals/eos/v2.0/cleos/command-reference/get/actions
        let full_txjson = {};
        full_txjson.actions = [];

        try {
          while (tx_nbhours < required_hours_tx_history) {
            //this.debugStatus = this.debugStatus.concat("\n\n New API Call with pos=", cpos);
            let partial_txjson = await this.wax.rpc.history_get_actions(this.wax.userAccount, cpos, -nb_tx_tofetch_per_apicall);
            //this.debugStatus = txjson;

            for (let k = partial_txjson.actions.length - 1; k >= 0; k--) {
              //We decrement because the JSON is returned with older TX first,
              //so that we have an easier processing, and diagnosis if needed (recent first)
              full_txjson.actions.push(partial_txjson.actions[k]);
              //full_txjson.actions.unshift(partial_txjson.actions);
            }

            //checking how many hours of transaction history we have
            let oldest_tx_datetime = new Date(partial_txjson.actions[0].block_time + "Z");
            tx_nbhours = (current_datetime - oldest_tx_datetime) / (3600 * 1000);
            cpos = partial_txjson.actions[0].account_action_seq;

            //this.debugStatus = this.debugStatus.concat("\n oldest_tx_datetime : ", oldest_tx_datetime, " # action_seq = ", cpos);
            //this.debugStatus = this.debugStatus.concat("\n tx_history_time : ", tx_nbhours);
            //this.debugStatus = this.debugStatus.concat("\n nbtotal_fetched_tx : ", partial_txjson.actions.length);
          }

          //this.debugStatus = this.debugStatus.concat("\n\n FINAL JSON TX nb : ", full_txjson.actions.length);
          //this.debugStatus = this.debugStatus.concat("\n\n", JSON.stringify(full_txjson, null, 2));

        }
        catch (e) {
          this.actionStatus = this.actionStatus.concat("\nCould not get Blockchain history : ", e);
        }


        this.actionStatus = this.actionStatus.concat("\n- Fetched ", full_txjson.actions.length, " transactions (~", tx_nbhours.toFixed(2), " hours of history)");
        
        let harvest_wait_time_sec = 60; //default wait time in seconds

        if (full_txjson.actions.length > 0) {
          //We could get some data history from the blockchain, keep going...

          let current_asset_id = ""; //Var for assigning asset_id
          let current_asset_harvest_timing = 10000; //Var for assigning harvest timing (minutes)
          let current_asset_type = ""; //Var for assigning asset_type
          let had_food_shortage = false; //if we have a food shortage during harvest, we will launch an immediate new cycle
          let had_water_shortage = false; //right now just for tracking, but no action

          await delay(500);
          //for (let i in assets) {
          for (let i = 0; i < assets.length; i++) {
            current_asset_id = assets[i][0];
            current_asset_harvest_timing = assets[i][1];
            current_asset_type = assets[i][2];
            let last_harvest_datetime;

            this.actionStatus = this.actionStatus.concat("\n- Checking ID #", current_asset_id, " - ", current_asset_type, " (", current_asset_harvest_timing, " min)");

            for (let j in full_txjson.actions) {
              //The transaction JSON is now sorted directly in the correct order (last transaction 1st) during the multi-step API call
              //we always process last transactions first, and we stop processing as soon as we found the last harvest time in the Blockchain (the most recent)

              let txa_trace = full_txjson.actions[j].action_trace;

              //looking only for farminggames > harvestanim or harvestbuild action, then if "asset_id" is matching
              //trying to determine when this animal (or asset) was last harvested
              if (txa_trace.act.account === "farminggames") {
                if (txa_trace.act.name.toLowerCase() === "harvestanim" || txa_trace.act.name.toLowerCase() === "harvestbuild") {
                  if (txa_trace.act.data.asset_id === current_asset_id) {
                    //We found the asset we were looking for, getting the last harvest datetime, then exiting for loop
                    //this.actionStatus = this.actionStatus.concat("\n\n FOUND ! ", txa_trace.block_time);
                    last_harvest_datetime = new Date(txa_trace.block_time + "Z");
                    break;
                  }
                } else if (txa_trace.act.name.toLowerCase() === "hvstmultanim" || txa_trace.act.name.toLowerCase() === "hvstmultbuild") {
                  //We found an multi-harvest call, we should look in asset_ids if we can find our ID
                  for (let x in txa_trace.act.data.asset_ids) {
                    if (txa_trace.act.data.asset_ids[x] === current_asset_id) {
                      last_harvest_datetime = new Date(txa_trace.block_time + "Z");
                      break;
                    }
                  }
                  //if last_harvest_datetime was set => we found a matching harvest timing => exiting main for to avoid overriding with older harvest data
                  if (last_harvest_datetime) {
                    break;
                  }
                }
              }
            }

            let next_harvest_datetime = new Date(last_harvest_datetime);

            //let next_harvest_datetime = new Date(last_harvest_datetime + current_asset_harvest_timing*60*1000);
            next_harvest_datetime.setMinutes(next_harvest_datetime.getMinutes() + current_asset_harvest_timing);

            //this.debugStatus = this.debugStatus.concat("\n\n Asset_ID : ", current_asset_id);
            //this.debugStatus = this.debugStatus.concat("\n- Last harvest Datetime : ", last_harvest_datetime);
            //this.debugStatus = this.debugStatus.concat("\n- Next harvest Datetime : ", next_harvest_datetime);
            //this.debugStatus = this.debugStatus.concat("\n- Current Datetime : ", current_datetime);

            //We only update optimal_minutes_towait if the current is lower
            if (((next_harvest_datetime - current_datetime)/1000) < optim_harvest_wait_time_sec ) {
              optim_harvest_wait_time_sec = Math.floor((next_harvest_datetime - current_datetime)/1000);
            }
            //this.debugStatus = this.debugStatus.concat("\n optimal_wait_time_sec : ", optimal_wait_time_sec);

            let harvest_needed = false;
            if (!last_harvest_datetime) {
              //if last_harvest_datetime is not set => we did not find any harvest history in the blockchain
              //if the required_hours_tx_history param is correctly set (max harvest timing) 
              //it could means it's a new stacked Asset (never harvested), and it should be ready for harvesting, so we try
              harvest_needed = true;
              this.actionStatus = this.actionStatus.concat(" : No history, let's try ! Harvesting...");
            } else {
              if (current_datetime > next_harvest_datetime) {
                harvest_needed = true;
                this.actionStatus = this.actionStatus.concat(" : ready ! Harvesting...");
                //this.debugStatus = this.debugStatus.concat("\n Asset is ready to be harvested !")
              } else {
                harvest_needed = false;
                this.actionStatus = this.actionStatus.concat(" : not ready...");
                //this.debugStatus = this.debugStatus.concat("\n Asset is NOT ready yet !")
              }
            }

            //harvest_needed = false; //Uncomment to force Disable Harvest for real (testing / diag purpose only)

            if (harvest_needed) {
              found_something_toharvest = true;
              if (!use_multi_harvest) {
                try {
                  if (current_asset_type.toString().toLowerCase().includes("animal")) {
                    //Harvesting an animal
                    multi_harvest_animal_ids.push(current_asset_id)
                    this.result = await this.wax.api.transact(
                      {
                        actions: [
                          {
                            account: "farminggames",
                            name: "harvestanim",
                            authorization: [
                              {
                                actor: this.wax.userAccount,
                                permission: "active",
                              },
                            ],
                            data: {
                              account: this.wax.userAccount,
                              asset_id: current_asset_id,
                            },
                          },
                        ],
                      },
                      {
                        blocksBehind: 3,
                        expireSeconds: 45,
                      }
                    );
                  } else if (current_asset_type.toString().toLowerCase().includes("building")) {
                    //Harvesting a building (Lumberjack, Pier, ...)
                    multi_harvest_building_ids.push(current_asset_id)
                    this.result = await this.wax.api.transact(
                      {
                        actions: [
                          {
                            account: "farminggames",
                            name: "harvestbuild",
                            authorization: [
                              {
                                actor: this.wax.userAccount,
                                permission: "active",
                              },
                            ],
                            data: {
                              account: this.wax.userAccount,
                              asset_id: current_asset_id,
                            },
                          },
                        ],
                      },
                      {
                        blocksBehind: 3,
                        expireSeconds: 45,
                      }
                    );
                  }
                  
                  this.actionStatus = this.actionStatus.concat(" DONE !");
                  harvested_something_withsuccess = true;
                  await delay(2000);

                } catch (e) {
                  this.actionStatus = this.actionStatus.concat("\nCould not Harvest : ", e);
                  console.log(e);

                  //if (e == "Error: assertion failure with message: Not enough food to harvest") {
                  if (e.toString().toLowerCase().includes("not enough food")) {
                    this.actionStatus = this.actionStatus.concat("\n- Refilling food....");
                    try {
                      this.result = await this.wax.api.transact(
                        {
                          actions: [
                            {
                              account: "farminggames",
                              name: "refillfood",
                              authorization: [
                                {
                                  actor: this.wax.userAccount,
                                  permission: "active",
                                },
                              ],
                              data: {
                                account: this.wax.userAccount,
                              },
                            },
                          ],
                        },
                        {
                          blocksBehind: 3,
                          expireSeconds: 45,
                        }
                      );
                      i = i - 1; //We setup i back 1 to harvest the same asset again
                      this.actionStatus = this.actionStatus.concat(" DONE !");
                      this.actionStatus = this.actionStatus.concat("\n- Trying harvesting the same asset again...");
                      had_food_shortage = true;
                    } catch (e) {
                      //We could harvest, but encountered an error while refilling food
                      //We set a 60 seconds cooldown before going on...
                      this.actionStatus = this.actionStatus.concat("\nCould not refill food : ", e);
                      this.actionStatus = this.actionStatus.concat("\nLet's cooldown for 60 seconds");
                      await delay(60*1000);
                    }
                  } else if (e.toString().toLowerCase().includes("not enough resources")) {
                    //"not enough resources" can concern both water AND food on some buildings (Lumberjack)
                    //So we start with water. Could be improved by reading real values of water/food stock in the blockchain
                    if (!had_water_shortage) {
                      //We did not refill water on this cycle, so we start here
                      this.actionStatus = this.actionStatus.concat("\n- Refilling water....");
                      try {
                        this.result = await this.wax.api.transact(
                          {
                            actions: [
                              {
                                account: "farminggames",
                                name: "refillwater",
                                authorization: [
                                  {
                                    actor: this.wax.userAccount,
                                    permission: "active",
                                  },
                                ],
                                data: {
                                  account: this.wax.userAccount,
                                },
                              },
                            ],
                          },
                          {
                            blocksBehind: 3,
                            expireSeconds: 45,
                          }
                        );
                        i = i - 1; //We setup i back 1 to harvest the same asset again
                        this.actionStatus = this.actionStatus.concat(" DONE !");
                        this.actionStatus = this.actionStatus.concat("\n- Trying harvesting the same asset again...");
                        had_water_shortage = true;
                      } catch (e) {
                        //We could harvest, but encountered an error while refilling food
                        //We set a 60 seconds cooldown before going on...
                        this.actionStatus = this.actionStatus.concat("\nCould not refill water : ", e);
                        this.actionStatus = this.actionStatus.concat("\nLet's cooldown for 60 seconds");
                        await delay(60*1000);

                      } 
/*
Since 2021-12-27, harvesting lumberjack / fishing pier does not require food anymore, but only water
So we can disable this part of the code

                    } else if (!had_food_shortage) {
                      //Water was not refilled on this cycle, neither food, so we continue with that option
                      this.actionStatus = this.actionStatus.concat("\n- Refilling food....");
                      try {
                        this.result = await this.wax.api.transact(
                          {
                            actions: [
                              {
                                account: "farminggames",
                                name: "refillfood",
                                authorization: [
                                  {
                                    actor: this.wax.userAccount,
                                    permission: "active",
                                  },
                                ],
                                data: {
                                  account: this.wax.userAccount,
                                },
                              },
                            ],
                          },
                          {
                            blocksBehind: 3,
                            expireSeconds: 45,
                          }
                        );
                        i = i - 1; //We setup i back 1 to harvest the same asset again
                        this.actionStatus = this.actionStatus.concat(" DONE !");
                        this.actionStatus = this.actionStatus.concat("\n- Trying harvesting the same asset again...");
                        had_food_shortage = true;
                      } catch (e) {
                        //We could harvest, but encountered an error while refilling food
                        //We set a 60 seconds cooldown before going on...
                        this.actionStatus = this.actionStatus.concat("\nCould not refill food : ", e);
                        this.actionStatus = this.actionStatus.concat("\nLet's cooldown for 60 seconds");
                        await delay(60*1000);
                      }
*/

                    }
                  } else {
                    //We encountered a unknown error while harvesting, most likely this one
                    //TypeError: (intermediate value) is not iterable
                    //We set a 60 seconds cooldown before going on...
                    this.actionStatus = this.actionStatus.concat("\nSomething went wront while harvesting this asset, let's cooldown for 60 seconds");
                    await delay(60*1000);
                  }

                  await delay(2*1000);
                  continue;
                }
              } else {
                //We are using multi harvest
                if (current_asset_type.toString().toLowerCase().includes("animal")) {
                  //Harvesting an animal
                  multi_harvest_animal_ids.push(current_asset_id)
                  this.actionStatus = this.actionStatus.concat(" will be done using multi-harvest");

                } else if (current_asset_type.toString().toLowerCase().includes("building")) {
                  //Harvesting a building (Lumberjack, Pier, ...)
                  multi_harvest_building_ids.push(current_asset_id)
                  this.actionStatus = this.actionStatus.concat(" will be done using multi-harvest");
                }
              }
            }
          }

          //We have screened  all assets in this cycle
          //Checking if we have to use the Harvest all function
          if (use_multi_harvest) { 
            try {
              //Harvesting multiple animals list is not empty, let's multi harvest
              if (multi_harvest_animal_ids.length > 0) {
                this.actionStatus = this.actionStatus.concat("\n- Harvesting multiple animals....");
                this.actionStatus = this.actionStatus.concat("\nlist : ", multi_harvest_animal_ids);
                this.result = await this.wax.api.transact(
                  {
                    actions: [
                      {
                        account: "farminggames",
                        name: "hvstmultanim",
                        authorization: [
                          {
                          actor: this.wax.userAccount,
                          permission: "active",
                          },
                        ],
                        data: {
                          account: this.wax.userAccount,
                          asset_ids: multi_harvest_animal_ids,
                        },
                      },
                    ],
                  },
                  {
                    blocksBehind: 3,
                    expireSeconds: 45,
                  }
                );
                this.actionStatus = this.actionStatus.concat(" DONE !");
                harvested_something_withsuccess = true;
                await delay(2000);
              }

              //Harvesting multiple buildings list is not empty, let's multi harvest
              if (multi_harvest_building_ids.length > 0) {
                this.actionStatus = this.actionStatus.concat("\n- Harvesting multiple buildings....");
                this.actionStatus = this.actionStatus.concat("\nlist : ", multi_harvest_building_ids);
                this.result = await this.wax.api.transact(
                  {
                    actions: [
                      {
                        account: "farminggames",
                        name: "hvstmultbuild",
                        authorization: [
                          {
                            actor: this.wax.userAccount,
                            permission: "active",
                          },
                        ],
                        data: {
                          account: this.wax.userAccount,
                          asset_ids: multi_harvest_building_ids,
                        },
                      },
                    ],
                  },
                  {
                    blocksBehind: 3,
                    expireSeconds: 45,
                  }
                );
                this.actionStatus = this.actionStatus.concat(" DONE !");
                harvested_something_withsuccess = true;
                await delay(2000);
              }

            } catch (e) {
              console.log(e);
              this.actionStatus = this.actionStatus.concat("\nCould not Harvest : ", e);
              this.actionStatus = this.actionStatus.concat("\nLet's cooldown for 60 seconds");
              await delay(60*1000);
            }
          }

          //Setting the best timing for next Harvest Cycle
          if (had_food_shortage) {
            //If we encountered a food shortage and had to refill, the current Asset may have not been harveted.
            //For that reason we launch a new cycle almost right away
            harvest_wait_time_sec = 3; //3 seconds wait is enough
            this.actionStatus = this.actionStatus.concat("\nThis cycle ended.. Starting a new cycle right now as we refilled food during this Cycle");
          } else {
            if (harvested_something_withsuccess) {
              //We harvested in that cycle, we use the custom setup for wait time
              //Edit on 2021-12-04 : we were using previously found_something_toharvest but new items are harvested to often (as they trigger harvest trial)
              harvest_wait_time_sec = Math.floor(Math.random() * (std_harvest_max_wait_time_sec - std_harvest_min_wait_time_sec + 1) + std_harvest_min_wait_time_sec);
              this.actionStatus = this.actionStatus.concat("\nThis cycle ended.. Ready for a next cycle in ", harvest_wait_time_sec, " seconds");
            } else {
              //We did NOT harvested in that cycle, we use the optimizes setup for wait time
              harvest_wait_time_sec = optim_harvest_wait_time_sec;
              harvest_wait_time_sec += Math.floor(Math.random() * (40 - 10 + 1) + 10); //adding a few more seconds (random 10s-40s) for safety
              this.actionStatus = this.actionStatus.concat("\nThis cycle ended.. We had nothing to do this time... Let's optimize and wait for ", harvest_wait_time_sec, " seconds");
            }
          }
        } else {
          //OUPS ! Something went wront while fetching Blockchain history, let's cooldown
          this.actionStatus = this.actionStatus.concat("\nSomething went wront while fetching Blockchain history, let's cooldown for 300 seconds");
          harvest_wait_time_sec = 300 //Blockchain issue => waiting for 5 minutes
        }
        await delay(1000*harvest_wait_time_sec);
      }
    },
  },
};

/*
############################ CHANGELOG ####################################


2021-12-27 :
- Updated the Lumberjack and Fishing Pier harvest timing (from 30 to 60 minutes)
- Lumberjack and Fishing Pier harvest now need only water (and no food) => remove the food refill code section in that case


2021-12-23 : 
Some updates to adapt to the new ingame full CBIt / SEST harvest :
- Updated the food refill to use the new funtion "refillfood" instead of sending 5 CBIT with "food_refill" memo
- Updated the water refill to use the new funtion "refillwater" instead of sending 5 CBIT with "water_refill" memo
Don't forget to send your SEST / CBIT ingame before using


2021-12-16 :
A lot of updates to support the new (partial at the moment) CBIT usage for refilling food and water
- Introduced a new "use_multi_harvest" setting, as this feature still use SEST (and not CBIT)
- Blockchain history now also looks in hvstmultanim / hvstmultbuild history to determine last harvest datetime for assets


2021-12-14 : 
- Fishingpier harvest added, tested, OK (same process as Lumberjack => was fine by just adding the ID and type in the table)


2021-12-08 : 
- Lumberjack harvest added
- handling "not enough resources" error message when harvesting Lumberjack (refilling water first, then food if repeated)


###################### TECHNICAL RESOURCES ################################

Useless for real users, just some info to keep an eye one for dev

//chain.wax.io doest not support history transaction (?)
//wax: new waxjs.WaxJS("https://chain.wax.io", null, null, false),
//wax: new waxjs.WaxJS("https://wax.greymass.com", null, null, false),
//https://api.wax.greeneosio.com

Useful technique resources :
https://github.com/greymass/eosio-core/blob/master/src/api/v1/history.ts
https://developers.eos.io/manuals/eos/v2.0/cleos/command-reference/get/actions

API calls for WAX history :
https://wax.greymass.com/v1/history/get_actions

Common errors while harvesting :
Error: assertion failure with message: Not enough food to harvest
TypeError: (intermediate value) is not iterable
Error: billed CPU time (240 us) is greater than the maximum billable CPU time for the transaction (0 us)
Error: assertion failure with message: Not enough resources to harvest 
=> be careful with this last one when harvesting Lumberjack it can be either Water OR Food missing

*/
</script>


<style>
.login-block {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>