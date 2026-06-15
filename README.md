const {
  EmbedBuilder,
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
  StringSelectMenuBuilder,
  ModalBuilder,
  TextInputBuilder,
  TextInputStyle,
  ChannelType,
  PermissionFlagsBits,
  UserSelectMenuBuilder
} = require('discord.js');
const fs = require('fs');
const path = require('path');
const discordTranscripts = require('discord-html-transcripts');

const dataDir = path.join(__dirname, '../../data');
if (!fs.existsSync(dataDir)) fs.mkdirSync(dataDir);
const SAYAC_PATH = path.join(dataDir, 'ticketSayac.json');

function sayacOku() { try { return JSON.parse(fs.readFileSync(SAYAC_PATH, 'utf8')); } catch { return {}; } }
function sayacYaz(data) { fs.writeFileSync(SAYAC_PATH, JSON.stringify(data, null, 2)); }

function getTicketSayac(kategori) {
  const sayac = sayacOku();
  const anahtar = kategori.toLowerCase();
  sayac[anahtar] = (sayac[anahtar] || 0) + 1;
  sayacYaz(sayac);
  return sayac[anahtar];
}

async function dmLogGonder(interaction, config, hedef, embed, durum) {
    const logKanalId = config.channels?.dm_log || config.dm_log;
    const logKanal = interaction.guild.channels.cache.get(logKanalId);
    if (!logKanal) return;

    const logo = interaction.guild.iconURL({ dynamic: true });
    const tarihSaat = new Date().toLocaleString('tr-TR');

    const logEmbed = new EmbedBuilder()
        .setColor(durum ? '#2ecc71' : '#e74c3c')
        .setAuthor({ name: 'LyxoraNW • DM Sistemi', iconURL: logo })
        .setTitle(durum ? '✅ DM Başarıyla Gönderildi' : '❌ DM Gönderilemedi')
        .setThumbnail(hedef.user.displayAvatarURL({ dynamic: true }))
        .setDescription(`Bir kullanıcıya özel mesaj gönderildi.\n\n**Durum:** ${durum ? '✅ Mesaj gönderildi' : '❌ Mesaj gönderilemedi'}`)
        .addFields(
            { name: '👤 Alıcı', value: `${hedef.user.username}`, inline: true },
            { name: '👮 Yetkili', value: `${interaction.user.username}`, inline: true },
            { name: '📝 İçerik', value: `\`\`\`${embed.data.description || 'İçerik Görüntülenemiyor'}\`\`\``, inline: false }
        )
        .setFooter({ text: tarihSaat });

    await logKanal.send({ embeds: [logEmbed] }).catch(() => {});
}

function yetkiKontrol(interaction, config) {
    const yetkiliRoller = [
        config.roles.destek_ekibi,
        config.roles.moderator,
        config.roles.admin,
        config.roles.yonetici,
        config.roles.kurucu,
        config.roles.sunucu_sahibi
    ];
    return interaction.member.roles.cache.some(r => yetkiliRoller.includes(r.id));
}

// 1. ANA PANEL SETUP
async function ticketSetup(kanal, config) {
  const logo = kanal.guild.iconURL({ dynamic: true });
  const embed = new EmbedBuilder()
    .setColor('#FF9900')
    .setAuthor({ name: 'Lyxora Network | Destek Sistemi', iconURL: logo })
    .setTitle('LYXORA NETWORK - DESTEK PANELİ')
    .setThumbnail(logo) 
    .setDescription(
      '## 🎫 **BİLGİLENDİRME**\n' +
      'Destek taleplerine aşağıdaki menü üzerinden ulaşabilirsin. **Uygun seçeneği seçerek ticket oluşturabilirsin.**\n' +
      'Tüm talepler **yetkililer tarafından** sırayla incelenir ve en kısa sürede dönüş yapılır.\n\n' +
      '## 📌 **NOT**\n' +
      'Lütfen sabırlı olun. **Yetkilileri etiketlemek yasaktır**, gereksiz ticket açmak yasaktır ve yanlış kategori seçilirse **kapatılabilir**.\n\n' +
      '## ⏰ **MESAİ SAATLERİ**\n' +
      'Destek talepleri **09:00 - 19:00** saatleri arasında işleme alınır.\n' +
      'Mesai saatleri dışında başvurular, **bir sonraki mesai gününde** işleme alınacaktır.'
    );

  const selectMenu = new StringSelectMenuBuilder()
    .setCustomId('ticket_sec')
    .setPlaceholder('Bir talep türünü seçin...')
    .addOptions([
        { label: 'Site-Ödeme Destek', description: 'VIP, ödeme and satın alma sorunları için destek.', value: 'odeme', emoji: '💳' },
        { label: 'Oyuncu Hile Bildirim', description: 'Hile kullanan oyuncuları bildirmek için destek.', value: 'hile', emoji: '⚔️' },
        { label: 'Yetkili Şikayet', description: 'Yetkililer hakkında şikayet oluşturmak için destek.', value: 'yetkili_sikayet', emoji: '📋' },
        { label: 'Oyuncu Şikayet', description: 'Kuralları ihlal eden oyuncuları bildirmek için destek.', value: 'oyuncu_sikayet', emoji: '🚨' },
        { label: 'Bug Bildirim', description: 'Sunucudaki hataları bildirmek için destek.', value: 'bug', emoji: '🐞' },
        { label: 'Ödül Alma', description: 'Kazandığınız ödülü almak için destek.', value: 'odul', emoji: '🎉' },
        { label: 'İade', description: 'Sunucu hatalarından kaynaklı kayıplar için destek.', value: 'iade', emoji: '🧾' },
        { label: 'Medya Rolü', description: 'Medya rolü başvurusu ve bilgi için destek.', value: 'medya-basvuru', emoji: '🎥' },
        { label: 'Youtuber Rolü', description: 'YouTuber başvurusu ve rol başvurusu için destek.', value: 'youtuber-basvuru', emoji: '📺' },
        { label: 'Diğer', description: 'Hiçbir kategoriye uymayan talepler için destek.', value: 'diger', emoji: '❓' }
    ]);

  await kanal.send({ 
    embeds: [embed], 
    components: [new ActionRowBuilder().addComponents(selectMenu)] 
  });
}

async function ticketAc(interaction, config) {
    const mevcutKanal = interaction.guild.channels.cache.find(c => c.topic && c.topic.startsWith(`ticket:${interaction.user.id}`));
    if (mevcutKanal) return interaction.reply({ content: `İzin verilen maksimum bilet sayısına (1) ulaştınız. <#${mevcutKanal.id}>`, flags: 64 });

    const kategori = interaction.values;

    // Discord API'sinin en güncel kuralına göre düzenlenen payload
    const rawModalPayload = {
        title: '🎫 Destek Talebi Formu',
        custom_id: `ticket_modal_api:${kategori}`,
        components: [
            {
                type: 1, // ActionRow 1
                components: [
                    {
                        type: 4, // TextInput
                        custom_id: 'mc_kullanici_adi',
                        label: 'Minecraft Kullanıcı Adınız:',
                        style: 1, // Short
                        placeholder: 'Örn: LyxoraPlayer',
                        min_length: 3,
                        max_length: 16,
                        required: true
                    }
                ]
            },
            {
                type: 1, // ActionRow 2
                components: [
                    {
                        type: 4, // TextInput
                        custom_id: 'ticket_sebep',
                        label: 'Sorununuzu açıklayın:',
                        style: 2, // Paragraph
                        placeholder: 'Örn: Yaşadığım şu sorun hakkında...',
                        min_length: 5,
                        max_length: 200,
                        required: true
                    }
                ]
            },
            {
                type: 18, // Label/Section Bileşeni
                label: 'Talebinizin Öncelik Durumu:',
                description: 'Lütfen biletinizin aciliyet seviyesini aşağıdan seçin.',
                // DİKKAT: Hata veren yer düzeltildi. "components" yerine "component" (Tekil nesne) yapıldı!
                component: {
                    type: 3, // StringSelectMenu (Açılır Menü)
                    custom_id: 'ticket_oncelik',
                    placeholder: 'Bir öncelik seviyesi seçin...',
                    min_values: 1,
                    max_values: 1,
                    options: [
                        { label: 'Düşük 🟢', value: 'low', description: 'Acil olmayan durumlar' },
                        { label: 'Normal 🟡', value: 'medium', description: 'Genel sorular ve sorunlar' },
                        { label: 'Yüksek 🔴', value: 'high', description: 'Kritik ve acil durumlar' }
                    ]
                }
            }
        ]
    };

    await interaction.showModal(rawModalPayload).catch((err) => console.error("Modal Gösterme Hatası:", err));
}

async function ticketModalOnaylaAPI(interaction, config) {
  try {
    if (!interaction.deferred && !interaction.replied) {
        await interaction.deferReply({ flags: 64 }); 
    }

    const kategoriHam = interaction.customId.split(':');
    
    const mcAdi = interaction.fields.getTextInputValue('mc_kullanici_adi');
    const sebep = interaction.fields.getTextInputValue('ticket_sebep');
    
    // Kullanıcının menüden seçtiği öncelik değerini alıp küçük harfe çeviriyoruz
    let oncelikYazi = 'normal';
    try {
        const selectValues = interaction.fields.getSelectMenuValues('ticket_oncelik');
        if (selectValues && selectValues.length > 0) oncelikYazi = selectValues;
    } catch {
        try {
            for (const row of interaction.components) {
                const comp = row.components.find(c => c.customId === 'ticket_oncelik');
                if (comp && comp.value) { oncelikYazi = comp.value; break; }
                else if (comp && comp.values && comp.values.length > 0) { oncelikYazi = comp.values; break; }
            }
        } catch {}
    }
    
    // Discord API'sinin ham data katmanından modal select menu değerini kesin olarak çeken garanti blok
    try {
        if (interaction.data && interaction.data.components) {
            for (const row of interaction.data.components) {
                if (row.components) {
                    const menuComp = row.components.find(c => c.custom_id === 'ticket_oncelik');
                    if (menuComp && menuComp.values && menuComp.values.length > 0) {
                        oncelikYazi = menuComp.values;
                        break;
                    }
                }
            }
        }
    } catch {}
    
    // Dizi olarak gelen veriyi parantezlerden kurtarıp saf metne çeviriyoruz
    if (Array.isArray(oncelikYazi)) {
        oncelikYazi = oncelikYazi[0];
    } else if (typeof oncelikYazi === 'object') {
        oncelikYazi = Object.values(oncelikYazi)[0];
    }
    oncelikYazi = String(oncelikYazi || 'normal').toLowerCase().trim();
    
    let emoji = '🟡';
    let oncelikMetni = 'Normal (Standart)';

    // Kullanıcının yazdığı metne veya menü seçimine göre esnek kontrol yapıyoruz
    if (oncelikYazi.includes('yüksek') || oncelikYazi.includes('yuksek') || oncelikYazi.includes('acil') || oncelikYazi.includes('high')) {
        emoji = '🔴';
        oncelikMetni = 'Yüksek (Acil Müdahale)';
    } else if (oncelikYazi.includes('düşük') || oncelikYazi.includes('dusuk') || oncelikYazi.includes('bekle') || oncelikYazi.includes('low')) {
        emoji = '🟢';
        oncelikMetni = 'Düşük (Bekleyebilirim)';
    }

    const biletNo = getTicketSayac(kategoriHam[1] || 'genel');
    const logo = interaction.guild.iconURL({ dynamic: true });
    const tarihSaat = new Date().toLocaleString('tr-TR');

    const permissionOverwrites = [
      { id: interaction.guild.id, deny: [PermissionFlagsBits.ViewChannel] },
      { id: interaction.user.id, allow: [PermissionFlagsBits.ViewChannel, PermissionFlagsBits.SendMessages, PermissionFlagsBits.ReadMessageHistory] }
    ];

    const yetkiliRoller = [
      config.roles?.destek_ekibi,
      config.roles?.moderator,
      config.roles?.admin,
      config.roles?.yonetici,
      config.roles?.kurucu,
      config.roles?.sunucu_sahibi
    ];

    yetkiliRoller.forEach(roleId => {
      if (roleId && typeof roleId === 'string' && roleId.trim() !== "" && interaction.guild.roles.cache.has(roleId)) {
        permissionOverwrites.push({
          id: roleId,
          allow: [PermissionFlagsBits.ViewChannel, PermissionFlagsBits.SendMessages, PermissionFlagsBits.ReadMessageHistory]
        });
      }
    });

    const kanal = await interaction.guild.channels.create({
      name: `${emoji}-${kategoriHam}-${biletNo}`,
      type: ChannelType.GuildText,
      parent: config.ticket.kategori,
      topic: `ticket:${interaction.user.id}:${mcAdi}:${oncelikMetni}`,
      permissionOverwrites: permissionOverwrites
    });

    const ticketLogKanalId = config.ticket?.ticket_log || config.ticket_log || config.channels?.ticket_log;
    const ticketLogKanal = interaction.guild.channels.cache.get(ticketLogKanalId);
    
    if (ticketLogKanal) {
        const acilisLogEmbed = new EmbedBuilder()
            .setColor('#2ecc71')
            .setAuthor({ name: 'LyxoraNetwork | Ticket Sistemi Logu', iconURL: logo })
            .setTitle('✨ Yeni Destek Talebi Açıldı')
            .setThumbnail(interaction.user.displayAvatarURL({ dynamic: true }))
            .addFields(
                { name: '👤 Talebi Açan', value: `${interaction.user}\n(\`${interaction.user.username}\`)`, inline: true },
                { name: '🎮 Minecraft Adı', value: `\`${mcAdi}\``, inline: true },
                { name: '📁 Kategori & Bilet', value: `\`${kategoriHam}-${biletNo}\``, inline: true },
                { name: '⚡ Öncelik Durumu', value: `${emoji} ${oncelikMetni}`, inline: false },
                { name: '📍 Açılan Kanal', value: `${kanal} (\`#${kanal.name}\`)`, inline: false },
                { name: '📝 Talep Sebebi', value: `\`\`\`${sebep}\`\`\``, inline: false }
            )
            .setFooter({ text: `${tarihSaat}` });

        await ticketLogKanal.send({ embeds: [acilisLogEmbed] }).catch(() => {});
    }

    const biletEmbed = new EmbedBuilder()
      .setColor('#FF9900')
      .setAuthor({ name: `${interaction.guild.name}`, iconURL: logo })
      .setTitle(`🎫 Yeni Destek Talebi #${biletNo}`)
      .setThumbnail(logo)
      .setDescription(
        `👋 Merhaba ${interaction.user},\n` +
        `Destek ekibi en kısa sürede ilgilenecek.\n\n` +

        `📌 **Bilgiler**\n` +
        `👤 Kullanıcı: ${interaction.user}\n` +
        `⚡ Öncelik: ${emoji} ${oncelikMetni}\n` +
        `📂 Kategori: ${kategoriHam}\n` +
        `\n` +
        `🎮 **Minecraft Bilgisi**\n` +
        `MC Adı: ${mcAdi}\n\n` +

        `📝 **Açıklama**\n` +
        `${sebep}`
      )
      .setFooter({ text: 'Yetkililer en kısa sürede dönüş yapacaktır' })
      .setTimestamp();

    const actionRow = new ActionRowBuilder().addComponents(
      new ButtonBuilder().setCustomId('ticket_kapat_onay_istek').setLabel('Bileti Kapat').setStyle(ButtonStyle.Danger).setEmoji('🔒'),
      new ButtonBuilder().setCustomId('ticket_ustlen').setLabel('Bileti Üstlen').setStyle(ButtonStyle.Success).setEmoji('✋'),
      new ButtonBuilder().setCustomId('ticket_uye_ekle_menu').setLabel('Üye Ekle').setStyle(ButtonStyle.Primary).setEmoji('👤')
    );

    const ekibeEtiket = config.roles?.destek_ekibi ? `<@&${config.roles.destek_ekibi}>` : "@Destek Ekibi";
    await kanal.send({ content: `${interaction.user} | ${ekibeEtiket}`, embeds: [biletEmbed], components: [actionRow] });
    
    await interaction.editReply({ content: `✅ Biletiniz başarıyla açıldı: ${kanal}`, embeds: [], components: [] });

  } catch (e) {
    console.error("Bilet Açma Hatası:", e);
    if (!interaction.replied && !interaction.deferred) { await interaction.reply({ content: '❌ Kanal oluşturulurken veya bilet açılırken bir hata meydana geldi.', flags: 64 }).catch(() => {}); } else { await interaction.followUp({ content: '❌ Kanal oluşturulurken veya bilet açılırken bir hata meydana geldi.', flags: 64 }).catch(() => {}); }
  }
}

async function ticketKapatOnayPanel(interaction, config) {
    if (!yetkiKontrol(interaction, config)) return interaction.reply({ content: '❌ Bu işlemi sadece destek ekibi yapabilir!', flags: 64 });

        const embed = new EmbedBuilder()
            .setColor('#FF9900')
            .setTitle('⚠️ Emin misin?')
            .setDescription('Bileti kapatmak üzeresin. Bu işlem sonucunda kanal silinecektir.');

        const row = new ActionRowBuilder().addComponents(
            new ButtonBuilder().setCustomId('ticket_kapat_onay').setLabel('Evet').setStyle(ButtonStyle.Success),
            new ButtonBuilder().setCustomId('ticket_iptal').setLabel('İptal').setStyle(ButtonStyle.Danger)
        );

        await interaction.reply({ embeds: [embed], components: [row] });
}

async function ticketKapat(interaction, config) {
    if (!yetkiKontrol(interaction, config)) return interaction.reply({ content: '❌ Bu işlemi sadece destek ekibi yapabilir!', flags: 64 });

        if (!interaction.deferred && !interaction.replied) await interaction.deferReply();

        try {
            const kapatmaSebep = interaction.fields.getTextInputValue('kapat_sebep');
            const kanal = interaction.channel;
            const topicParcalar = (kanal.topic || '').split(':');
            const userId = topicParcalar[1];
            const mcAdi = topicParcalar[2] || 'Belirtilmedi';
            const sahibi = await interaction.guild.members.fetch(userId).catch(() => null);

            const transcript = await discordTranscripts.createTranscript(kanal, {
                limit: -1,
                fileName: `transcript-${kanal.name}.html`,
                saveImages: true,
                poweredBy: false
            });

            const logo = interaction.guild.iconURL({ dynamic: true });
            const ticketIdBul = kanal.name.match(/\d+/);
            const ticketIdNo = ticketIdBul ? ticketIdBul[0] : "1";

            const gunler = ["Pazar", "Pazartesi", "Salı", "Çarşamba", "Perşembe", "Cuma", "Cumartesi"];
            const aylar = ["Ocak", "Şubat", "Mart", "Nisan", "Mayıs", "Haziran", "Temmuz", "Ağustos", "Eylül", "Ekim", "Kasım", "Aralık"];
            const simdi = new Date();
            const formatliTarihGorsel = `${simdi.getDate()} ${aylar[simdi.getMonth()]} ${simdi.getFullYear()} ${gunler[simdi.getDay()]} ${String(simdi.getHours()).padStart(2, '0')}:${String(simdi.getMinutes()).padStart(2, '0')}`;

            const ticketLogKanalId = config.ticket?.ticket_log || config.ticket_log || config.channels?.ticket_log;
            const ticketLogKanal = interaction.guild.channels.cache.get(ticketLogKanalId);

            await interaction.editReply({ content: '🔒 Bilet başarıyla kapatıldı, kanal 5 saniye içinde siliniyor...' });

            if (ticketLogKanal) {
                const depoKanalId = config.channels?.dokum_log || config.channels?.dm_log || config.dm_log;
                const depoKanal = interaction.guild.channels.cache.get(depoKanalId) || ticketLogKanal;
                
                const etiketliMetin = sahibi ? `<@${sahibi.id}> kullanıcısının bilet dökümü` : `Bilinmeyen bir kullanıcının bilet dökümü`;
                const depoMesaj = await depoKanal.send({ content: etiketliMetin, files: [transcript] }).catch(() => null);
                const transcriptUrl = depoMesaj?.attachments.first()?.url || "https://play.lyxoranw.com.tr";

                const logKapatEmbed = new EmbedBuilder()
                    .setColor('#e74c3c')
                    .setAuthor({ name: 'LyxoraNetwork | Ticket Sistemi Logu', iconURL: logo })
                    .setTitle('🚩 Bilet Kapatıldı ve Arşivlendi')
                    .setThumbnail(interaction.user.displayAvatarURL({ dynamic: true }))
                    .addFields(
                        { name: '👮 Kapatan Yetkili', value: `${interaction.user} (\`${interaction.user.username}\`)`, inline: true },
                        { name: '👤 Bilet Sahibi', value: sahibi ? `${sahibi.user} (\`${sahibi.user.username}\`)` : 'Bilinmiyor', inline: true },
                        { name: '🎮 Minecraft Adı', value: `\`${mcAdi}\``, inline: true },
                        { name: '🆔 Bilet Bilgisi', value: `\`No: ${ticketIdNo} - ${kanal.name}\``, inline: false },
                        { name: '📝 Kapatma Sebebi', value: `\`\`\`${kapatmaSebep}\`\`\``, inline: false }
                    ).setFooter({ text: simdi.toLocaleString('tr-TR') });

                const downloadRow = new ActionRowBuilder().addComponents(
                    new ButtonBuilder()
                        .setLabel('🗄️ Transcript Görüntüle')
                        .setURL(transcriptUrl)
                        .setStyle(ButtonStyle.Link)
                );

                await ticketLogKanal.send({ embeds: [logKapatEmbed], components: [downloadRow] }).catch(() => {});

                if (sahibi) {
                    const dmEmbed = new EmbedBuilder()
                        .setColor('#e74c3c')
                        .setAuthor({ name: `${interaction.guild.name}`, iconURL: logo })
                        .setTitle('🔒 Ticket Kapatıldı')
                        .setThumbnail(interaction.user.displayAvatarURL({ dynamic: true }))
                        .setDescription('Destek talebiniz başarıyla kapatıldı.')
                        .addFields(
                            { name: '📅 Kapatılma Tarihi', value: formatliTarihGorsel, inline: true },
                            { name: '🆔 Ticket ID', value: ticketIdNo, inline: true },
                            { name: '👤 Tarafından Kapatıldı', value: interaction.user.username, inline: true },
                            { name: '📝 Kapatma Sebebi', value: kapatmaSebep, inline: false }
                        )
                        .setFooter({ text: `Teşekkür ederiz! • bugün saat ${String(simdi.getHours()).padStart(2, '0')}:${String(simdi.getMinutes()).padStart(2, '0')}` });

                    const dmDurum = await sahibi.send({ embeds: [dmEmbed], components: [downloadRow] }).then(() => true).catch(() => false);
                    await dmLogGonder(interaction, config, sahibi, dmEmbed, dmDurum);
                }
            }

            setTimeout(() => kanal.delete().catch(() => {}), 5000);

        } catch (error) {
            console.error("Kapatma Hatası:", error);
            if (!interaction.replied && !interaction.deferred) { await interaction.reply({ content: '❌ Kanal kapatılırken veya döküm oluşturulurken bir hata oluştu.', flags: 64 }).catch(() => {}); } else { await interaction.followUp({ content: '❌ Kanal kapatılırken veya döküm oluşturulurken bir hata oluştu.', flags: 64 }).catch(() => {}); }
        }
    }

    async function ticketUstlen(interaction, config) {
        if (!yetkiKontrol(interaction, config)) return interaction.reply({ content: '❌ Bu işlemi sadece destek ekibi yapabilir!', flags: 64 });

        const actionRow = new ActionRowBuilder().addComponents(
            new ButtonBuilder().setCustomId('ticket_kapat_onay_istek').setLabel('Bileti Kapat').setStyle(ButtonStyle.Danger).setEmoji('🔒'), 
            new ButtonBuilder().setCustomId('ticket_ustlendi').setLabel('Üstlenildi').setStyle(ButtonStyle.Secondary).setDisabled(true),
            new ButtonBuilder().setCustomId('ticket_uye_ekle_menu').setLabel('Üye Ekle').setStyle(ButtonStyle.Primary).setEmoji('👤')
        );
        await interaction.message.edit({ components: [actionRow] });
        
        const embed = new EmbedBuilder()
            .setColor('#5865F2')
            .setDescription(`Biletiniz ${interaction.user} tarafından üstlenilmiştir.`);

        await interaction.reply({ embeds: [embed] });
    }

    async function ticketKapatOnay(interaction, config) {
        if (!yetkiKontrol(interaction, config)) return interaction.reply({ content: '❌ Bu işlemi sadece destek ekibi yapabilir!', flags: 64 });

        const modal = new ModalBuilder().setCustomId('ticket_kapat_modal').setTitle('🔒 Bileti Kapat');
        modal.addComponents(new ActionRowBuilder().addComponents(new TextInputBuilder().setCustomId('kapat_sebep').setLabel('Kapatma Sebebi:').setPlaceholder('Bilet neden kapatılıyor?').setStyle(TextInputStyle.Paragraph).setRequired(true)));
        await interaction.showModal(modal);
    }

    async function ticketUyeEkleMenu(interaction, config) {
        if (!yetkiKontrol(interaction, config)) return interaction.reply({ content: '❌ Bu işlemi sadece destek ekibi yapabilir!', flags: 64 });

        const userSelect = new UserSelectMenuBuilder().setCustomId('ticket_uye_ekle_final').setPlaceholder('Eklenecek üyeyi seç...').setMinValues(1).setMaxValues(1);
        await interaction.reply({ content: '👤 **Talebe eklenecek üyeyi seç**', components: [new ActionRowBuilder().addComponents(userSelect)], flags: 64 });
    }

    async function ticketUyeEkleFinal(interaction) {
        const targetId = interaction.values[0];
        const targetUser = await interaction.guild.members.fetch(targetId);
        await interaction.channel.permissionOverwrites.create(targetUser, { ViewChannel: true, SendMessages: true, ReadMessageHistory: true });
        
        const embed = new EmbedBuilder()
            .setColor('#00FF00')
            .setDescription(`✅ ${targetUser} başarıyla talebe eklendi.`);

        await interaction.reply({ embeds: [embed] });
    }

    module.exports = { 
        ticketSetup, ticketAc, ticketModalOnaylaAPI, 
        ticketKapat, ticketUstlen, ticketKapatOnay, ticketKapatOnayPanel,
        ticketUyeEkleMenu, ticketUyeEkleFinal
    };
