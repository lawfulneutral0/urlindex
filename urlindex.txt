
var CFG = {
    scanDelay: 1200,      
    monitorPoll: 1000,    
    uiInterval: 10000,    

    minReaction: 34000,   
    maxReaction: 158000,   

    historyLimit: 25,
    maxAgeDays: 14,
    keywords: ["giveaway", "win", "prize", "winner", "ends in", "ends:", "hosted by", "hosted:", "event", "click", "enter"],
    
    blacklist: [ "selfbot", "fake", "spam", "automate", "program", "honeypot" ],
    channelWhitelist: ["giveaway", "drop", "event", "prize", "gift", "loot"]
};

var STATE = { 
    scanning: false, 
    monitoring: true, 
    stats: { scanned: 0, joined: 0 }, 
    msgId: null, 
    cache: {},
    queue: {} 
};


function sleep(ms) { ethone.sleep(ms * (0.9 + Math.random() * 0.3)); }
function nonce() { return Date.now().toString() + Math.floor(Math.random() * 99999).toString(); }
function log(m) { ethone.log("[Lawful] " + m); }
function randomDelay(min, max) { return Math.floor(Math.random() * (max - min + 1) + min); }

function safeClone(obj) {
    try {
        if (!obj) return null;
        return JSON.parse(JSON.stringify(obj));
    } catch (e) {
        return null;
    }
}

function extractText(msg) {
    if (!msg) return "";
    var txt = (msg.content || "").toLowerCase();
    
    try {
        if (msg.embeds && msg.embeds.length > 0) {
            for (var i = 0; i < msg.embeds.length; i++) {
                var e = msg.embeds[i];
                if (!e) continue;
                txt += " " + (e.title || "") + " " + (e.description || "") + " " + (e.footer ? e.footer.text : "") + " " + (e.author ? e.author.name : "");
                if (e.fields && e.fields.length > 0) {
                    for (var j = 0; j < e.fields.length; j++) {
                        var f = e.fields[j];
                        if (f) txt += " " + (f.name || "") + " " + (f.value || "");
                    }
                }
            }
        }
    } catch(e) {}
    return txt.toLowerCase();
}

function fetchHistory(id, limit) {
    var url = "https://discord.com/api/v10/channels/" + id + "/messages?limit=" + limit;
    var headers = { "Accept-Encoding": "identity", "Content-Type": "application/json" };
    var res = discord.http_get(url, headers);
    if (res.status === 200 && res.body) {
        try { return JSON.parse(res.body); } catch (e) { return null; }
    }
    return null;
}

function joinButton(guildId, channelId, msg, btn) {
    try {
        var payload = JSON.stringify({
            type: 3, nonce: nonce(), guild_id: guildId, channel_id: channelId,
            message_flags: 0, message_id: msg.id, application_id: msg.author.id,
            session_id: nonce(), data: { component_type: 2, custom_id: btn.custom_id }
        });
        var res = discord.http_post("https://discord.com/api/v10/interactions", payload);
        return (res.status >= 200 && res.status < 300);
    } catch(e) { return false; }
}

function executeJoin(guildId, channelId, msg) {
    if (STATE.cache[msg.id]) return false;
    var done = false;

    try {
        if (msg.components && msg.components.length > 0) {
            for (var i = 0; i < msg.components.length; i++) {
                var row = msg.components[i];
                if (row.components) {
                    for (var j = 0; j < row.components.length; j++) {
                        var btn = row.components[j];
                        if (btn.type !== 2 || btn.disabled) continue;
                        
                        var valid = false;
                        if (btn.emoji && (btn.emoji.name === "🎉" || btn.emoji.name === "🎁")) valid = true;
                        
                        var bid = (btn.custom_id || "").toLowerCase();
                        var blabel = (btn.label || "").toLowerCase();
                        if (!valid && (bid.includes("enter") || bid.includes("join") || blabel.includes("enter") || blabel.includes("join"))) valid = true;

                        if (valid) {
                            if (joinButton(guildId, channelId, msg, btn)) {
                                done = true;
                                STATE.cache[msg.id] = true;
                            }
                            sleep(1000);
                        }
                    }
                }
            }
        }
    } catch(e) {}

    try {
        if (!done && msg.reactions) {
            for (var r = 0; r < msg.reactions.length; r++) {
                var rx = msg.reactions[r];
                if (rx.emoji.name === "🎉") {
                    if (rx.me) continue;
                    var eid = rx.emoji.id ? (rx.emoji.name + ":" + rx.emoji.id) : rx.emoji.name;
                    if (discord.add_reaction(channelId, msg.id, eid)) {
                        done = true;
                        STATE.cache[msg.id] = true;
                    }
                    sleep(1000);
                }
            }
        }
    } catch(e) {}
    
    return done;
}

function handleIncoming(rawMsg, ctx) {
    try {
        if (!STATE.monitoring && !STATE.scanning) return; 

        var msg = safeClone(rawMsg);
        if (!msg) return; 

        var guildId = ctx.guild ? ctx.guild.id : (msg.guild_id || "");
        var channelId = ctx.channel ? ctx.channel.id : (msg.channel_id || "");
        var authorId = msg.author ? msg.author.id : "";

        if (authorId === ethone.user.id) return;
        if (STATE.cache[msg.id]) return; 

        var channelName = ctx.channel ? ctx.channel.name : "";
        if (channelName && CFG.channelWhitelist.length > 0) {
            var isTarget = false;
            var lowerName = channelName.toLowerCase();
            for (var w = 0; w < CFG.channelWhitelist.length; w++) {
                if (lowerName.includes(CFG.channelWhitelist[w])) { isTarget = true; break; }
            }
            if (!isTarget) return; 
        }

        var txt = extractText(msg);
        var isGw = false;
        
        for (var k = 0; k < CFG.keywords.length; k++) {
            if (txt.includes(CFG.keywords[k])) { isGw = true; break; }
        }
        
        if (isGw) {
            for (var b = 0; b < CFG.blacklist.length; b++) {
                if (txt.includes(CFG.blacklist[b])) { isGw = false; break; }
            }
        }

        if (isGw) {
            if (STATE.queue[msg.id]) {
                STATE.queue[msg.id].msg = msg; 
                log("[Monitor] Updated msg " + msg.id);
            } else {
                var reactionTime = randomDelay(CFG.minReaction, CFG.maxReaction);
                var gName = ctx.guild ? ctx.guild.name : "Server";
                
                STATE.queue[msg.id] = {
                    guildId: guildId,
                    channelId: channelId,
                    msg: msg, 
                    processAt: Date.now() + reactionTime,
                    name: gName
                };
                log("[Monitor] Queued: " + gName + " (Wait " + (reactionTime/1000).toFixed(0) + "s)");
            }
        }
    } catch (e) {}
}

function processQueue() {
    var keys = Object.keys(STATE.queue);
    var now = Date.now();

    for (var i = 0; i < keys.length; i++) {
        var key = keys[i];
        var item = STATE.queue[key];
        
        if (item && now >= item.processAt) {
            var success = executeJoin(item.guildId, item.channelId, item.msg);

            if (!success) {
                var fresh = fetchHistory(item.channelId, 5);
                if (fresh && Array.isArray(fresh)) {
                    for(var f=0; f<fresh.length; f++) {
                        if (fresh[f].id === item.msg.id) {
                            success = executeJoin(item.guildId, item.channelId, fresh[f]);
                            break;
                        }
                    }
                }
            }

            if (success) {
                STATE.stats.joined++;
                ethone.send_toast("Joined: " + item.name, "success");
            }
            delete STATE.queue[key];
        }
    }
}

discord.on_message = function(ctx) { 
    try { handleIncoming(ctx.message, ctx); } catch(e){} 
};
discord.on_message_update = function(ctx) { 
    try { if(ctx.after) handleIncoming(ctx.after, ctx); } catch(e){} 
};

function updateUI(ctx, statusMsg, guildName, progress) {
    if (!statusMsg) return;
    try {
        var content = "```ini\n[ ⚖️ Lawful Giveaway Scan ]\n\n";
        content += "[ Status ]  Scanning : " + (guildName || "In Progress...") + "\n";
        content += "[ Prog   ]  " + progress + "\n";
        content += "[ Stats  ]  Joined: " + STATE.stats.joined + "  |  Scanned: " + STATE.stats.scanned + "\n";
        content += "```";
        discord.edit_message(ctx.channel.id, statusMsg.id, content);
    } catch(e) {}
}

function runMonitorLoop(ctx) {
    if (STATE.monitoring) return discord.send_message(ctx.channel.id, "⚠️ **Monitor Active**");
    
    STATE.monitoring = true;
    discord.send_message(ctx.channel.id, "🕵️ **Monitor Started**");
    
    try {
        while (STATE.monitoring) {
            processQueue();
            sleep(CFG.monitorPoll); 
        }
    } catch(e) { log("Mon Error: " + e); }
    STATE.monitoring = false;
}

function runScan(ctx) {
    if (STATE.scanning) return discord.send_message(ctx.channel.id, "⚠️ **Scan Active**");
    STATE.scanning = true;
    var msg = discord.send_message(ctx.channel.id, "🚀 **Scanning...**");
    var nextUpdate = 0;
    
    var guilds = discord.fetch_guilds();
    if (!guilds) return;

    try {
        for (var g = 0; g < guilds.length; g++) {
            if (!STATE.scanning) break;
            var guild = guilds[g];

            if (Date.now() > nextUpdate) {
                updateUI(ctx, msg, guild.name, (g+1)+"/"+guilds.length);
                nextUpdate = Date.now() + 8000;
            }
            processQueue();

            var channels = discord.fetch_guild_channels(guild.id);
            if (!channels) { sleep(2000); continue; }

            for (var c = 0; c < channels.length; c++) {
                if (!STATE.scanning) break;
                processQueue();

                var ch = channels[c];
                if (ch.type !== 0 && ch.type !== 5) continue;
                
                var isTarget = (CFG.channelWhitelist.length === 0);
                for(var w=0; w<CFG.channelWhitelist.length; w++) if(ch.name.includes(CFG.channelWhitelist[w])) isTarget=true;
                if (!isTarget) continue;

                STATE.stats.scanned++;
                var msgs = fetchHistory(ch.id, CFG.historyLimit);
                if (msgs && Array.isArray(msgs)) {
                    for (var m = 0; m < msgs.length; m++) {
                        if ((Date.now() - Date.parse(msgs[m].timestamp)) > (CFG.maxAgeDays * 86400000)) continue;
                        
                        var clone = safeClone(msgs[m]); // Clone history too
                        var txt = extractText(clone);
                        var isGw = false;
                        for(var k=0; k<CFG.keywords.length; k++) if(txt.includes(CFG.keywords[k])) isGw=true;
                        if(isGw) {
                             if(executeJoin(guild.id, ch.id, clone)) {
                                 STATE.stats.joined++;
                                 ethone.send_toast("Joined: " + guild.name, "success");
                                 sleep(2000);
                             }
                        }
                    }
                }
                sleep(CFG.scanDelay);
            }
        }
    } catch(e) { log("Err: " + e); }

    STATE.scanning = false;
    discord.edit_message(ctx.channel.id, msg.id, "✅ Done. Joined: " + STATE.stats.joined);
}

ethone.on_command("scanforgiveaways", "Scan", "scanforgiveaways", function(ctx) { runScan(ctx); });
ethone.on_command("stopscan", "Stop", "stopscan", function(ctx) { 
    STATE.scanning = false; 
    discord.send_message(ctx.channel.id, "🛑 Stopping...");
});
ethone.on_command("togglemonitor", "Toggle", "togglemonitor", function(ctx) { 
    if (STATE.monitoring) { 
        STATE.monitoring = false; 
        discord.send_message(ctx.channel.id, "🛑 Stopping Monitor..."); 
    } else { 
        runMonitorLoop(ctx); 
    }
});

ethone.log("BetterGiveaways by Lawful Neutral Loaded");
