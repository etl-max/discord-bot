// ============================================================
// 🤖 BOT DISCORD COMPLET V2 — TOUTES FONCTIONNALITÉS
// ============================================================

const {
  Client, GatewayIntentBits, PermissionFlagsBits, ChannelType,
  EmbedBuilder, ActionRowBuilder, ButtonBuilder, ButtonStyle,
  StringSelectMenuBuilder, Events
} = require("discord.js");

// ────────────────────────────────────────────────────────────
// ⚙️ CONFIGURATION — REMPLIS CES VALEURS
// ────────────────────────────────────────────────────────────
const BOT_TOKEN      = "MTUyNDM4MzQ5Njg2MzQ4MjAxNw.G3RFSs.iu9EmCPVCcShTuKIQsCg00kpHyaSokJB9pugRc";
const GUILD_ID       = "1524372238734725330";
const PREFIX         = "+";
const STAFF_ROLES    = ["Modérateur", "Admin", "Staff", "Haut Gradé"];

// ────────────────────────────────────────────────────────────
// DONNÉES EN MÉMOIRE
// ────────────────────────────────────────────────────────────
const blacklist     = new Map(); // userId → { reason, lastMsg, lastVoc }
const jailed        = new Map(); // userId → { jailChannelId }
const mutedUsers    = new Map(); // userId → { until, reason }
const msgCount      = new Map(); // userId → count
const spamTracker   = new Map(); // userId → [timestamps]
const spamStrikes   = new Map(); // userId → strikes (0=mute, 1=ban1sem, 2=ban def)
const verifiedUsers = new Set(); // userIds vérifiés
const joinTracker   = [];        // timestamps des joins récents

// ────────────────────────────────────────────────────────────
// CLIENT
// ────────────────────────────────────────────────────────────
const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMembers,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
    GatewayIntentBits.GuildVoiceStates,
    GatewayIntentBits.GuildPresences,
    GatewayIntentBits.GuildInvites,
    GatewayIntentBits.GuildModeration,
  ],
});

const wait = (ms) => new Promise((r) => setTimeout(r, ms));

// ────────────────────────────────────────────────────────────
// UTILITAIRES
// ────────────────────────────────────────────────────────────
function isStaff(member) {
  return member.permissions.has(PermissionFlagsBits.ManageMessages) ||
    member.roles.cache.some((r) => STAFF_ROLES.includes(r.name));
}

function isHighGrade(member) {
  return member.permissions.has(PermissionFlagsBits.Administrator) ||
    member.roles.cache.some((r) => ["Admin", "Haut Gradé"].includes(r.name));
}

function parseDuration(str) {
  if (!str) return null;
  const match = str.match(/^(\d+)(s|m|h|j|sem)$/i);
  if (!match) return null;
  const val  = parseInt(match[1]);
  const unit = match[2].toLowerCase();
  const map  = { s: 1000, m: 60000, h: 3600000, j: 86400000, sem: 604800000 };
  return val * (map[unit] || 0);
}

function formatDuration(ms) {
  if (ms < 60000)      return `${Math.floor(ms / 1000)} seconde(s)`;
  if (ms < 3600000)   return `${Math.floor(ms / 60000)} minute(s)`;
  if (ms < 86400000)  return `${Math.floor(ms / 3600000)} heure(s)`;
  if (ms < 604800000) return `${Math.floor(ms / 86400000)} jour(s)`;
  return `${Math.floor(ms / 604800000)} semaine(s)`;
}

async function getOrCreateRole(guild, name, color = "#555555") {
  let role = guild.roles.cache.find((r) => r.name === name);
  if (!role) role = await guild.roles.create({ name, color, mentionable: false });
  return role;
}

// ────────────────────────────────────────────────────────────
// SALON DE LOGS (privé, hauts gradés uniquement)
// ────────────────────────────────────────────────────────────
async function getLogChannel(guild) {
  let ch = guild.channels.cache.find(
    (c) => c.name === "📋logs-staff" && c.type === ChannelType.GuildText
  );
  if (!ch) {
    const staffRole = guild.roles.cache.find((r) => STAFF_ROLES.includes(r.name));
    ch = await guild.channels.create({
      name: "📋logs-staff",
      type: ChannelType.GuildText,
      permissionOverwrites: [
        { id: guild.roles.everyone, deny: [PermissionFlagsBits.ViewChannel] },
        ...(staffRole
          ? [{ id: staffRole.id, allow: [PermissionFlagsBits.ViewChannel, PermissionFlagsBits.SendMessages] }]
          : []),
      ],
    });
  }
  return ch;
}

async function log(guild, embed) {
  try {
    const ch = await getLogChannel(guild);
    await ch.send({ embeds: [embed] });
  } catch (_) {}
}

function logEmbed(title, description, color = 0x5865f2, fields = []) {
  const e = new EmbedBuilder()
    .setColor(color)
    .setTitle(title)
    .setDescription(description)
    .setTimestamp();
  if (fields.length) e.addFields(fields);
  return e;
}

// ────────────────────────────────────────────────────────────
// VÉRIFICATION DES NOUVEAUX MEMBRES
// ────────────────────────────────────────────────────────────
const verificationMessages = new Map(); // userId → messageId

client.on(Events.GuildMemberAdd, async (member) => {
  const guild = member.guild;
  const now   = Date.now();

  // ── Anti-raid ──
  joinTracker.push(now);
  const recent = joinTracker.filter((t) => now - t < 10000);
  if (recent.length >= 8) {
    const logCh = await getLogChannel(guild);
    logCh.send({
      embeds: [logEmbed(
        "🚨 ALERTE RAID DÉTECTÉE",
        `**${recent.length}** membres ont rejoint en moins de 10 secondes !\nVérifiez immédiatement.`,
        0xff0000
      )],
    });
    try {
      await member.timeout(600000, "Anti-raid automatique");
    } catch (_) {}
  }

  // ── Vérification ──
  let verifCh = guild.channels.cache.find(
    (c) => c.name === "✅vérification" && c.type === ChannelType.GuildText
  );
  if (!verifCh) {
    verifCh = await guild.channels.create({
      name: "✅vérification",
      type: ChannelType.GuildText,
      permissionOverwrites: [
        { id: guild.roles.everyone, deny: [PermissionFlagsBits.ViewChannel] },
        { id: member.id, allow: [PermissionFlagsBits.ViewChannel, PermissionFlagsBits.SendMessages] },
      ],
    });
  } else {
    await verifCh.permissionOverwrites.edit(member.id, {
      ViewChannel: true,
      SendMessages: true,
    });
  }

  const btn = new ActionRowBuilder().addComponents(
    new ButtonBuilder()
      .setCustomId(`verify_${member.id}`)
      .setLabel("✅ Je suis humain — Vérifier")
      .setStyle(ButtonStyle.Success)
  );

  const verifEmbed = new EmbedBuilder()
    .setColor(0x5865f2)
    .setTitle("🤖 Vérification requise")
    .setDescription(
      `Bienvenue ${member} !\n\n` +
      `Pour accéder au serveur, clique sur le bouton ci-dessous pour prouver que tu n'es pas un bot.\n\n` +
      `*Si tu ne te vérifies pas dans 10 minutes, tu seras expulsé.*`
    );

  const msg = await verifCh.send({ embeds: [verifEmbed], components: [btn] });
  verificationMessages.set(member.id, msg.id);

  setTimeout(async () => {
    if (!verifiedUsers.has(member.id)) {
      try {
        await member.kick("Non vérifié dans les 10 minutes");
        log(guild, logEmbed("👢 Expulsion auto", `${member.user.tag} n'a pas vérifié son compte.`, 0xff9900));
      } catch (_) {}
    }
  }, 600000);

  sendWelcome(member, guild);

  log(guild, logEmbed(
    "📥 Nouveau membre",
    `${member.user.tag} (\`${member.id}\`) a rejoint le serveur.`,
    0x00ff00,
    [{ name: "Compte créé le", value: member.user.createdAt.toLocaleDateString("fr-FR"), inline: true }]
  ));
});

client.on(Events.InteractionCreate, async (interaction) => {
  if (!interaction.isButton()) return;

  if (interaction.customId.startsWith("verify_")) {
    const userId = interaction.customId.split("_")[1];
    if (interaction.user.id !== userId) {
      return interaction.reply({ content: "❌ Ce bouton n'est pas pour toi.", ephemeral: true });
    }

    verifiedUsers.add(userId);
    const memberRole = interaction.guild.roles.cache.find((r) => r.name === "Membre") ||
      await getOrCreateRole(interaction.guild, "Membre", "#3498db");

    try {
      await interaction.member.roles.add(memberRole);
      await interaction.reply({ content: "✅ Vérification réussie ! Bienvenue sur le serveur.", ephemeral: true });
      log(interaction.guild, logEmbed("✅ Vérification", `${interaction.user.tag} s'est vérifié.`, 0x00ff00));
    } catch (_) {}
  }

  if (interaction.customId === "close_ticket") {
    const ch = interaction.channel;
    await ch.send("🔒 Ticket fermé. Suppression dans 5 secondes...");
    setTimeout(() => ch.delete().catch(() => {}), 5000);
  }
});

// ────────────────────────────────────────────────────────────
// BIENVENUE CRÉATIVE
// ────────────────────────────────────────────────────────────
const welcomeStyles = [
  (m) => `☠️ *Un nouveau condamné arrive...* **${m.user.username}** vient de rejoindre. Le destin t'a mené ici... ou peut-être la malédiction.`,
  (m) => `💀 Les portes s'ouvrent... **${m.user.username}** entre dans l'ombre. Bienvenue parmi nous... si tu oses rester.`,
  (m) => `😂 MDR **${m.user.username}** vient d'atterrir ici comme une patate dans une soupe ! Bienvenue dans le chaos 🎉`,
  (m) => `💅 Ah, **${m.user.username}** daigne enfin nous rejoindre. On t'attendait... *soupir*. Bienvenue quand même.`,
  (m) => `🎭 **${m.user.username}** vient d'apparaître ! Le serveur était tranquille sans toi... mais bon, bienvenue !`,
  (m) => `🕯️ *Les bougies s'éteignent...* **${m.user.username}** vient de rejoindre. Personne ne sort d'ici indemne.`,
  (m) => `🌑 *Le silence se brise...* **${m.user.username}** a osé franchir le seuil. Bienvenue dans l'obscurité.`,
  (m) => `🤡 **${m.user.username}** est là ! On attendait le clown de service... ah non c'est sympa en fait, bienvenue !`,
];

async function sendWelcome(member, guild) {
  const style = welcomeStyles[Math.floor(Math.random() * welcomeStyles.length)];
  const ch = guild.channels.cache.find(
    (c) => c.type === ChannelType.GuildText &&
      (c.name.includes("bienvenue") || c.name.includes("accueil") || c.name.includes("général"))
  );
  if (!ch) return;

  const embed = new EmbedBuilder()
    .setColor(0x8b0000)
    .setTitle("👁️ Un nouveau membre...")
    .setDescription(style(member))
    .setThumbnail(member.user.displayAvatarURL())
    .setFooter({ text: `Membre #${guild.memberCount}` })
    .setTimestamp();

  ch.send({ embeds: [embed] });
}

// ────────────────────────────────────────────────────────────
// RELANCE HEBDOMADAIRE DES INACTIFS
// ────────────────────────────────────────────────────────────
async function weeklyInactiveCheck(guild) {
  await guild.members.fetch();
  const inactiveMembers = guild.members.cache.filter(
    (m) => !m.user.bot && (msgCount.get(m.id) || 0) < 10
  );

  if (inactiveMembers.size === 0) return;

  const messages = [
    (u) => `Hey ${u} ! On t'a pas beaucoup vu sur le serveur 👀 N'hésite pas à te lancer, tout le monde est sympa ici !`,
    (u) => `Coucou ${u} ! Le serveur serait encore mieux avec toi dedans 😄 Viens dire bonjour, on mord pas (trop) !`,
    (u) => `${u} tu nous manques ! 🥺 Viens discuter, faire des rencontres et t'amuser avec nous !`,
    (u) => `Yo ${u} ! T'es là mais on t'entend pas 😅 Lance-toi, le serveur t'attend les bras ouverts !`,
  ];

  let count = 0;
  for (const [, m] of inactiveMembers) {
    try {
      const msg = messages[Math.floor(Math.random() * messages.length)];
      await m.send({
        embeds: [new EmbedBuilder()
          .setColor(0x5865f2)
          .setTitle("👋 On pense à toi !")
          .setDescription(
            msg(m) + "\n\n" +
            `Rejoins-nous sur **${guild.name}** pour discuter, rigoler et ne pas te sentir seul(e) !\n` +
            `On est là pour toi 💙`
          )],
      });
      count++;
      await wait(1000);
    } catch (_) {}
  }

  const logCh = await getLogChannel(guild);
  logCh.send({
    embeds: [new EmbedBuilder()
      .setColor(0xf1c40f)
      .setTitle("📊 Rapport hebdomadaire — Membres inactifs")
      .setDescription(
        `**${inactiveMembers.size}** membres ont moins de 10 messages.\n` +
        `**${count}** messages de relance envoyés avec succès.`
      )
      .setTimestamp()],
  });
}

setInterval(() => {
  client.guilds.cache.get(GUILD_ID)?.let?.((g) => weeklyInactiveCheck(g));
}, 7 * 24 * 60 * 60 * 1000);

// ────────────────────────────────────────────────────────────
// ANTI-SPAM AVANCÉ
// ────────────────────────────────────────────────────────────
async function handleSpam(message) {
  const { member, guild, author } = message;
  const userId = author.id;
  const strikes = spamStrikes.get(userId) || 0;

  if (strikes === 0) {
    spamStrikes.set(userId, 1);
    try {
      await member.timeout(7 * 60 * 1000, "Anti-spam : 15 messages d'affilée");
    } catch (_) {}

    try {
      await author.send({
        embeds: [new EmbedBuilder()
          .setColor(0xff9900)
          .setTitle("⚠️ Avertissement Anti-Spam")
          .setDescription(
            `Tu as été **muté 7 minutes** pour spam.\n\n` +
            `Si tu continues après le mute, tu seras banni 1 semaine.\n` +
            `Si tu persistes encore, ce sera un **ban définitif**.`
          )],
      });
    } catch (_) {}

    message.channel.send({
      embeds: [logEmbed("🔇 Anti-Spam — Mute", `${author} muté 7 min pour spam.`, 0xff9900)],
    });
    log(guild, logEmbed("🔇 Anti-Spam", `${author.tag} muté 7 min (spam).`, 0xff9900));

  } else if (strikes === 1) {
    spamStrikes.set(userId, 2);
    try {
      await author.send({
        embeds: [new EmbedBuilder()
          .setColor(0xff0000)
          .setTitle("🔨 Ban temporaire — 1 semaine")
          .setDescription(
            `Tu as été banni **1 semaine** pour spam répété.\n\n` +
            `⚠️ Si tu recommences à ton retour, ce sera un **ban définitif**.`
          )],
      });
    } catch (_) {}

    await guild.bans.create(userId, { reason: "Anti-spam : récidive", deleteMessageSeconds: 86400 });
    setTimeout(async () => {
      try { await guild.bans.remove(userId, "Fin du ban 1 semaine"); } catch (_) {}
    }, 7 * 24 * 60 * 60 * 1000);

    log(guild, logEmbed("🔨 Anti-Spam — Ban 1 sem", `${author.tag} banni 1 semaine (spam récidive).`, 0xff0000));

  } else {
    try {
      await author.send({
        embeds: [new EmbedBuilder()
          .setColor(0x000000)
          .setTitle("💀 Ban définitif")
          .setDescription("Tu as été banni **définitivement** pour spam persistant. Au revoir.")],
      });
    } catch (_) {}

    await guild.bans.create(userId, { reason: "Anti-spam : ban définitif", deleteMessageSeconds: 604800 });
    log(guild, logEmbed("💀 Anti-Spam — Ban déf", `${author.tag} banni définitivement (spam persistant).`, 0x000000));
  }
}

// ────────────────────────────────────────────────────────────
// DÉTECTION DE VIRUS / LIENS MALVEILLANTS
// ────────────────────────────────────────────────────────────
const MALICIOUS_PATTERNS = [
  /discord\.gift\/[a-zA-Z0-9]+/i,
  /free.*nitro/i,
  /steamcommunity.*trade/i,
  /bit\.ly\//i,
  /tinyurl\.com\//i,
  /grabify\.link/i,
  /ngrok\.io/i,
  /@(everyone|here).*http/i,
  /\.exe\b/i,
  /\.bat\b/i,
  /token.*grabber/i,
  /ip.*logger/i,
];

function containsMalicious(content) {
  return MALICIOUS_PATTERNS.some((p) => p.test(content));
}

// ────────────────────────────────────────────────────────────
// ÉVÉNEMENT PRINCIPAL : MESSAGES
// ────────────────────────────────────────────────────────────
client.on(Events.MessageCreate, async (message) => {
  if (message.author.bot || !message.guild) return;

  const { guild, member, channel, author, content } = message;
  const userId = author.id;
  const now    = Date.now();

  msgCount.set(userId, (msgCount.get(userId) || 0) + 1);
  if (msgCount.get(userId) === 300) checkCustomRole(member, guild);

  if (containsMalicious(content)) {
    try { await message.delete(); } catch (_) {}
    channel.send({
      embeds: [logEmbed("🦠 Lien suspect supprimé", `${author} a envoyé un lien potentiellement dangereux.`, 0xff0000)],
    });
    log(guild, logEmbed("🦠 Lien suspect", `${author.tag} : \`${content.slice(0, 200)}\``, 0xff0000));
    return;
  }

  if (blacklist.has(userId)) {
    const bl = blacklist.get(userId);
    if (bl.lastMsg && now - bl.lastMsg < 120000) {
      try { await message.delete(); } catch (_) {}
      return;
    }
    bl.lastMsg = now;
    blacklist.set(userId, bl);
  }

  if (!spamTracker.has(userId)) spamTracker.set(userId, []);
  const timestamps = spamTracker.get(userId).filter((t) => now - t < 8000);
  timestamps.push(now);
  spamTracker.set(userId, timestamps);

  if (timestamps.length >= 15) {
    spamTracker.set(userId, []);
    await handleSpam(message);
    return;
  }

  if (!content.startsWith(PREFIX)) return;

  const args    = content.slice(PREFIX.length).trim().split(/ +/);
  const command = args.shift().toLowerCase();

  handleCommand(command, args, message);
});

// ────────────────────────────────────────────────────────────
// RÔLE PERSO À 300 MESSAGES
// ────────────────────────────────────────────────────────────
async function checkCustomRole(member, guild) {
  try {
    const ch = guild.channels.cache.find(
      (c) => c.type === ChannelType.GuildText && c.name.includes("général")
    );
    if (ch) {
      ch.send({
        content: `<@${member.id}>`,
        embeds: [new EmbedBuilder()
          .setColor(0xf1c40f)
          .setTitle("🎉 Félicitations !")
          .setDescription(
            `Tu as atteint **300 messages** !\n\n` +
            `Crée ton rôle perso avec :\n\`+monrole <nom> <couleur>\`\nEx: \`+monrole MonRôle #ff0000\``
          )],
      });
    }
  } catch (_) {}
}

// ────────────────────────────────────────────────────────────
// GESTION DES COMMANDES
// ────────────────────────────────────────────────────────────
async function handleCommand(command, args, message) {
  const { guild, member, channel, author } = message;

  if (command === "aide" || command === "help") {
    const embed = new EmbedBuilder()
      .setColor(0x5865f2)
      .setTitle("📋 Aide — Commandes du Bot")
      .setDescription("Voici toutes les commandes disponibles sur ce serveur.")
      .addFields(
        { name: "🛡️ Modération (Staff)", value:
          "`+mute @u <durée> <raison>` — Mute (30s/5m/2h/1j/1sem)\n" +
          "`+unmute @u` — Démute\n" +
          "`+kick @u <raison>` — Expulse\n" +
          "`+ban @u <raison>` — Ban définitif\n" +
          "`+clear <nb>` — Supprime des messages\n" +
          "`+jail @u <raison>` — Isolation\n" +
          "`+unjail @u` — Libère du jail\n" +
          "`+blacklist @u <raison>` — Blacklist\n" +
          "`+unblacklist @u` — Retire blacklist"
        },
        { name: "🎨 Rôles", value:
          "`+createrole <nom> <couleur>` — Crée un rôle\n" +
          "`+copierole @role <nom>` — Copie un rôle\n" +
          "`+monrole <nom> <couleur>` — Rôle perso (300 msgs)"
        },
        { name: "📊 Infos & Salons", value:
          "`+stats` — Stats du serveur\n" +
          "`+ticket` — Ouvre un ticket\n" +
          "`+invitation` — Génère un lien d'invitation permanent"
        },
        { name: "😂 Fun", value:
          "`+fakeban @u` — Faux ban avec compte à rebours\n" +
          "`+dmall <msg>` — DM tous les membres"
        },
        { name: "⚠️ Punitions — Explication", value:
          "**Mute** → Tu ne peux plus écrire pendant une durée définie.\n" +
          "**Jail** → Tu es isolé dans un salon privé, invisible partout.\n" +
          "**Blacklist** → 1 message / 2 min, 5 min max en vocal.\n" +
          "**Kick** → Expulsion du serveur (tu peux revenir).\n" +
          "**Ban** → Exclusion définitive, tu ne peux plus rejoindre.\n\n" +
          "**Anti-Spam** → 15 msgs rapides = mute 7min → ban 1sem → ban déf."
        }
      )
      .setFooter({ text: `Préfixe : ${PREFIX}` });

    channel.send({ embeds: [embed] });
    return;
  }

  // ==========================================
  // 1. SALONS & INFOS
  // ==========================================
  if (command === "stats") {
    const embed = new EmbedBuilder()
      .setColor(0x3498db)
      .setTitle(`📊 Stats de ${guild.name}`)
      .addFields(
        { name: "Membres", value: `${guild.memberCount}`, inline: true },
        { name: "Salons", value: `${guild.channels.cache.size}`, inline: true },
        { name: "Rôles", value: `${guild.roles.cache.size}`, inline: true }
      )
      .setTimestamp();
    channel.send({ embeds: [embed] });
    return;
  }

  if (command === "ticket") {
    let tCh = guild.channels.cache.find((c) => c.name === `ticket-${author.username}`);
    if (tCh) return message.reply(`❌ Tu as déjà un ticket ouvert : ${tCh}`);

    tCh = await guild.channels.create({
      name: `ticket-${author.username}`,
      type: ChannelType.GuildText,
      permissionOverwrites: [
        { id: guild.roles.everyone, deny: [PermissionFlagsBits.ViewChannel] },
        { id: author.id, allow: [PermissionFlagsBits.ViewChannel, PermissionFlagsBits.SendMessages] },
      ],
    });

    const row = new ActionRowBuilder().addComponents(
      new ButtonBuilder().setCustomId("close_ticket").setLabel("🔒 Fermer").setStyle(ButtonStyle.Danger)
    );
    await tCh.send({ content: `Bonjour ${author}, le staff arrive.`, components: [row] });
    message.reply(`✅ Ticket créé : ${tCh}`);
    return;
  }

  if (command === "clear") {
    if (!isStaff(member)) return message.reply("❌ Permission refusée.");
    const amount = parseInt(args[0]) || 10;
    let deleted = 0, remaining = amount;
    while (remaining > 0) {
      const toDelete = Math.min(remaining, 100);
      const msgs = await channel.messages.fetch({ limit: toDelete });
      if (msgs.size === 0) break;
      await channel.bulkDelete(msgs, true);
      deleted += msgs.size; remaining -= toDelete;
      await wait(1000);
    }
    const c = await channel.send({ embeds: [logEmbed("🗑️ Clear", `${deleted} messages supprimés.`, 0x00ff00)] });
    setTimeout(() => c.delete().catch(() => {}), 3000);
    log(guild, logEmbed("🗑️ Clear", `${author.tag} a supprimé ${deleted} messages dans #${channel.name}`, 0x00ff00));
    return;
  }

  if (command === "invitation" || command === "invite") {
    try {
      const publicChannel = guild.channels.cache.find(
        (c) => c.type === ChannelType.GuildText && c.permissionsFor(guild.roles.everyone).has(PermissionFlagsBits.ViewChannel)
      );
      if (!publicChannel) return message.reply("❌ Impossible de générer une invitation : aucun salon public trouvé.");

      const invite = await publicChannel.createInvite({
        maxAge: 0, 
        maxUses: 0, 
        reason: `Générée par ${author.tag} via la commande +invitation`
      });

      const embed = new EmbedBuilder()
        .setColor(0x00ff00)
        .setTitle("🔗 Lien d'invitation du serveur")
        .setDescription(`Voici le lien permanent pour inviter tes amis :\n\n**${invite.url}**`)
        .setFooter({ text: "Partage-le manuellement où tu le souhaites !" });

      channel.send({ embeds: [embed] });
    } catch (error) {
      message.reply("❌ Une erreur est survenue lors de la création du lien d'invitation.");
    }
    return;
  }

  // ==========================================
  // 2. RÔLES
  // ==========================================
  if (command === "createrole") {
    if (!isHighGrade(member)) return message.reply("❌ Permission refusée (Haut Gradé uniquement).");
    const name = args[0];
    const color = args[1] || "#99aab5";
    if (!name) return message.reply("❌ Spécifie un nom. Ex: `+createrole Test #ff0000`");
    const r = await guild.roles.create({ name, color, reason: `Créé par ${author.tag}` });
    message.reply(`✅ Rôle ${r} créé !`);
    return;
  }

  if (command === "copierole") {
    if (!isHighGrade(member)) return message.reply("❌ Permission refusée.");
    const source = message.mentions.roles.first() || guild.roles.cache.get(args[0]);
    const newName = args.slice(1).join(" ");
    if (!source || !newName) return message.reply("❌ Usage: `+copierole @role NouveauNom`");
    const r = await guild.roles.create({ name: newName, color: source.color, permissions: source.permissions.bitfield });
    message.reply(`✅ Rôle ${source.name} copié vers ${r} !`);
    return;
  }

  if (command === "monrole") {
    const count = msgCount.get(author.id) || 0;
    if (count < 300) return message.reply(`❌ Tu as besoin de 300 messages (Actuel: ${count}/300).`);
    const name = args[0];
    const color = args[1];
    if (!name || !color) return message.reply("❌ Usage: `+monrole MonNom #ff0000`");

    let r = guild.roles.cache.find((role) => role.name === `Perso-${author.username}`);
    if (!r) {
      r = await guild.roles.create({ name: `Perso-${author.username}`, color, reason: "Rôle perso 300 messages" });
      await member.roles.add(r);
    } else {
      await r.edit({ name, color });
    }
    message.reply(`✅ Ton rôle perso a été configuré : ${r}`);
    return;
  }

  // ==========================================
  // 3. MODÉRATION STRICte
  // ==========================================
  if (command === "kick") {
    if (!isStaff(member)) return message.reply("❌ Permission refusée.");
    const target = message.mentions.members.first();
    if (!target) return message.reply("❌ Mentionne un membre.");
    const reason = args.slice(1).join(" ") || "Comportement inapproprié";
    const level  = reason.length > 50 ? "⛔ ÉLEVÉ" : reason.length > 20 ? "⚠️ MODÉRÉ" : "🟡 FAIBLE";

    try {
      await target.send({ embeds: [new EmbedBuilder().setColor(0xff9900)
        .setTitle(`👢 Expulsé de **${guild.name}**`)
        .setDescription(`**Raison :** ${reason}\n**Niveau :** ${level}\n\n**Pour revenir :**\n• Lis le règlement\n• Respecte les membres\n• Présente tes excuses\n• Contacte un modérateur`)] });
    } catch (_) {}

    await target.kick(reason);
    message.reply({ embeds: [logEmbed("👢 Kick", `${target.user.tag} expulsé — ${reason} (${level})`, 0xff9900)] });
    log(guild, logEmbed("👢 Kick", `${author.tag} a expulsé ${target.user.tag} — ${reason}`, 0xff9900));
    return;
  }

  if (command === "ban") {
    if (!isStaff(member)) return message.reply("❌ Permission refusée.");
    const target = message.mentions.members.first();
    if (!target) return message.reply("❌ Mentionne un membre.");
    const reason = args.slice(1).join(" ") || "Violation grave des règles";

    try {
      await target.send({ embeds: [new EmbedBuilder().setColor(0xff0000)
        .setTitle(`🔨 Banni de **${guild.name}**`)
        .setDescription(`**Raison :** ${reason}\n\n**Ce ban est définitif.** Tu ne peux plus rejoindre ce serveur.`)] });
    } catch (_) {}

    await target.ban({ reason, deleteMessageSeconds: 604800 });
    message.reply({ embeds: [logEmbed("🔨 Ban", `${target.user.tag} banni — ${reason}`, 0xff0000)] });
    log(guild, logEmbed("🔨 Ban définitif", `${author.tag} a banni ${target.user.tag} — ${reason}`, 0xff0000));
    return;
  }

  if (command === "blacklist" || command === "bl") {
    if (!isStaff(member)) return message.reply("❌ Permission refusée.");
    const target = message.mentions.members.first();
    if (!target) return message.reply("❌ Mentionne un membre.");
    const reason = args.slice(1).join(" ") || "Comportement répété";
    blacklist.set(target.id, { reason, since: Date.now(), lastMsg: 0 });
    const blRole = await getOrCreateRole(guild, "Blacklisté", "#1a1a1a");
    await target.roles.add(blRole);

    try {
      await target.send({ embeds: [new EmbedBuilder().setColor(0x000000)
        .setTitle(`⛔ Blacklisté sur **${guild.name}**`)
        .setDescription(
          `**Raison :** ${reason}\n\n**Restrictions :**\n• 1 message toutes les 2 minutes\n• 5 minutes max en vocal\n\n` +
          `**Pour être retiré de la blacklist :**\n1. Envoie un MP à un Modérateur ou Haut Gradé\n` +
          `2. Explique pourquoi tu mérites d'être retiré\n3. Présente tes excuses sincères\n4. Attends la décision du staff`
        )] });
    } catch (_) {}

    message.reply({ embeds: [logEmbed("⛔ Blacklist", `${target} blacklisté — ${reason}`, 0x000000)] });
    log(guild, logEmbed("⛔ Blacklist", `${author.tag} a blacklisté ${target.user.tag} — ${reason}`, 0x000000));
    return;
  }

  if (command === "unblacklist" || command === "unbl") {
    if (!isStaff(member)) return message.reply("❌ Permission refusée.");
    const target = message.mentions.members.first();
    if (!target) return message.reply("❌ Mentionne un membre.");
    blacklist.delete(target.id);
    const blRole = guild.roles.cache.find((r) => r.name === "Blacklisté");
    if (blRole) await target.roles.remove(blRole);
    message.reply({ embeds: [logEmbed("✅ Unblacklist", `${target} retiré de la blacklist.`, 0x00ff00)] });
    try { await target.send({ embeds: [new EmbedBuilder().setColor(0x00ff00).setTitle(`✅ Plus blacklisté sur **${guild.name}**`).setDescription("Tes restrictions ont été levées. Respecte les règles.")] }); } catch (_) {}
    log(guild, logEmbed("✅ Unblacklist", `${author.tag} a retiré ${target.user.tag} de la blacklist`, 0x00ff00));
    return;
  }

  // ==========================================
  // 4. Mute / Isolation / Jail
  // ==========================================
  if (command === "mute") {
    if (!isStaff(member)) return message.reply("❌ Permission refusée.");
    const target = message.mentions.members.first();
    if (!target) return message.reply("❌ Mentionne un membre.");

    const dur    = parseDuration(args[1] || "10m");
    if (!dur) return message.reply("❌ Durée invalide. Ex: `30s`, `5m`, `2h`, `1j`, `1sem`");
    const reason = args.slice(2).join(" ") || "Aucune raison";

    const muteRole = await getOrCreateRole(guild, "Muet", "#808080");
    guild.channels.cache.forEach(async (ch) => {
      try { await ch.permissionOverwrites.edit(muteRole, { SendMessages: false, Speak: false, AddReactions: false }); } catch (_) {}
    });

    await target.roles.add(muteRole);
    mutedUsers.set(target.id, { until: Date.now() + dur, reason });

    message.reply({ embeds: [logEmbed("🔇 Mute", `${target} a été réduit au silence pour ${args[1] || "10m"}.`, 0xff9900)] });
    log(guild, logEmbed("🔇 Mute", `${author.tag} a muté ${target.user.tag} pendant ${args[1] || "10m"} — ${reason}`, 0xff9900));
    return;
  }

  if (command === "unmute") {
    if (!isStaff(member)) return message.reply("❌ Permission refusée.");
    const target = message.mentions.members.first();
    if (!target) return message.reply("❌ Mentionne un membre.");

    const muteRole = guild.roles.cache.find((r) => r.name === "Muet");
    if (muteRole) await target.roles.remove(muteRole);
    mutedUsers.delete(target.id);

    message.reply({ embeds: [logEmbed("🔊 Unmute", `${target} peut de nouveau parler.`, 0x00ff00)] });
    log(guild, logEmbed("🔊 Unmute", `${author.tag} a démuté ${target.user.tag}`, 0x00ff00));
    return;
  }

  if (command === "jail") {
    if (!isStaff(member)) return message.reply("❌ Permission refusée.");
    const target = message.mentions.members.first();
    if (!target) return message.reply("❌ Mentionne un membre.");
    const reason = args.slice(1).join(" ") || "Enquête en cours";

    let jailCh = guild.channels.cache.find((c) => c.name === `jail-${target.user.username}`);
    if (!jailCh) {
      jailCh = await guild.channels.create({
        name: `jail-${target.user.username}`,
        type: ChannelType.GuildText,
        permissionOverwrites: [
          { id: guild.roles.everyone, deny: [PermissionFlagsBits.ViewChannel] },
          { id: target.id, allow: [PermissionFlagsBits.ViewChannel, PermissionFlagsBits.SendMessages] }
        ]
      });
    }

    jailed.set(target.id, { jailChannelId: jailCh.id });
    message.reply(`🔒 ${target} a été isolé dans ${jailCh}`);
    log(guild, logEmbed("🔒 Jail", `${author.tag} a mis en jail ${target.user.tag} — ${reason}`, 0x555555));
    return;
  }

  if (command === "unjail") {
    if (!isStaff(member)) return message.reply("❌ Permission refusée.");
    const target = message.mentions.members.first();
    if (!target) return message.reply("❌ Mentionne un membre.");

    const data = jailed.get(target.id);
    if (data) {
      const ch = guild.channels.cache.get(data.jailChannelId);
      if (ch) await ch.delete().catch(() => {});
      jailed.delete(target.id);
    }

    message.reply(`🔓 ${target} a été libéré du jail.`);
    log(guild, logEmbed("🔓 Unjail", `${author.tag} a libéré ${target.user.tag}`, 0x00ff00));
    return;
  }

  // ==========================================
  // 5. LES AUTRES
  // ==========================================
  if (command === "fakeban") {
    const target = message.mentions.members.first();
    if (!target) return message.reply("❌ Mentionne le coupable.");
    const msg = await channel.send(`☣️ Initialisation du protocole de bannissement pour ${target}...`);
    await wait(1500); await msg.edit("🔄 Connexion aux serveurs Discord...");
    await wait(1500); await msg.edit("⚠️ Erreur système : Trop de sel détecté chez cet utilisateur.");
    await wait(1500); await msg.edit(`🤣 C'était une blague, ${target} a eu chaud !`);
    return;
  }

  if (command === "dmall") {
    if (!isHighGrade(member)) return message.reply("❌ Seuls les Hauts Gradés peuvent DM tout le monde.");
    const txt = args.join(" ");
    if (!txt) return message.reply("❌ Écris un message.");
    await guild.members.fetch();
    let ok = 0;
    for (const [, m] of guild.members.cache) {
      if (m.user.bot) continue;
      try { await m.send({ content: txt }); ok++; await wait(1000); } catch (_) {}
    }
    message.reply(`📢 Message envoyé en MP à **${ok}** membres.`);
    return;
  }
}

client.login(BOT_TOKEN);
