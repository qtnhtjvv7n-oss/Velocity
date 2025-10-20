const { Client, GatewayIntentBits, PermissionFlagsBits, EmbedBuilder } = require('discord.js');
const axios = require('axios');
const fs = require('fs');
const path = require('path');

const client = new Client({
    intents: [
        GatewayIntentBits.Guilds,
        GatewayIntentBits.GuildMessages,
        GatewayIntentBits.MessageContent,
        GatewayIntentBits.GuildMembers,
    ],
});

const config = require('./config.json');
const dataPath = path.join(__dirname, 'data.json');

let data = {
    blacklist: [],
    autorole: null,
    welcomeChannel: null
};

if (fs.existsSync(dataPath)) {
    data = JSON.parse(fs.readFileSync(dataPath, 'utf8'));
}

function saveData() {
    fs.writeFileSync(dataPath, JSON.stringify(data, null, 2));
}

function isStaff(member) {
    return member.permissions.has(PermissionFlagsBits.Administrator) || 
           member.permissions.has(PermissionFlagsBits.ManageGuild) ||
           member.permissions.has(PermissionFlagsBits.KickMembers) ||
           member.permissions.has(PermissionFlagsBits.BanMembers);
}

client.on('ready', () => {
    console.log(`✅ ${client.user.tag} is online!`);
    client.user.setActivity('Velocity Server', { type: 'WATCHING' });
});

client.on('guildMemberAdd', async (member) => {
    let channel = null;
    
    if (data.welcomeChannel) {
        channel = member.guild.channels.cache.get(data.welcomeChannel);
    }
    
    if (!channel) {
        channel = member.guild.systemChannel || 
                  member.guild.channels.cache.find(ch => ch.name === 'general') ||
                  member.guild.channels.cache.find(ch => {
                      if (!ch.isTextBased()) return false;
                      const perms = ch.permissionsFor(member.guild.members.me);
                      return perms && perms.has(PermissionFlagsBits.SendMessages);
                  });
    }
    
    if (channel) {
        const perms = channel.permissionsFor(member.guild.members.me);
        if (perms && perms.has(PermissionFlagsBits.SendMessages)) {
            try {
                await channel.send(`Welcome ${member} to velocity! Verify to gain access to everything else`);
            } catch (error) {
                console.error('Failed to send welcome message:', error);
            }
        }
    }
    
    if (data.autorole) {
        const role = member.guild.roles.cache.get(data.autorole);
        if (role) {
            member.roles.add(role).catch(console.error);
        }
    }
});

client.on('messageCreate', async (message) => {
    if (message.author.bot) return;
    if (!message.content.startsWith(config.prefix)) return;

    const args = message.content.slice(config.prefix.length).trim().split(/ +/);
    const command = args.shift().toLowerCase();

    if (data.blacklist.includes(message.author.id)) {
        return message.reply('❌ You are blacklisted from using bot commands.');
    }

    try {
        switch (command) {
            case 'ban':
                if (!isStaff(message.member)) {
                    return message.reply('❌ This command is staff only!');
                }
                if (!message.member.permissions.has(PermissionFlagsBits.BanMembers)) {
                    return message.reply('❌ You don\'t have permission to ban members!');
                }
                
                const userToBan = message.mentions.users.first();
                if (!userToBan) {
                    return message.reply('❌ Please mention a user to ban!');
                }
                
                const banReason = args.slice(1).join(' ') || 'No reason provided';
                
                let memberToBan;
                try {
                    memberToBan = await message.guild.members.fetch(userToBan.id);
                } catch (error) {
                    return message.reply('❌ User not found in this server!');
                }
                
                if (memberToBan.roles.highest.position >= message.member.roles.highest.position) {
                    return message.reply('❌ You cannot ban this user!');
                }
                
                await memberToBan.ban({ reason: banReason });
                message.reply(`✅ Successfully banned ${userToBan.tag} | Reason: ${banReason}`);
                break;

            case 'kick':
                if (!isStaff(message.member)) {
                    return message.reply('❌ This command is staff only!');
                }
                if (!message.member.permissions.has(PermissionFlagsBits.KickMembers)) {
                    return message.reply('❌ You don\'t have permission to kick members!');
                }
                
                const userToKick = message.mentions.users.first();
                if (!userToKick) {
                    return message.reply('❌ Please mention a user to kick!');
                }
                
                const kickReason = args.slice(1).join(' ') || 'No reason provided';
                
                let memberToKick;
                try {
                    memberToKick = await message.guild.members.fetch(userToKick.id);
                } catch (error) {
                    return message.reply('❌ User not found in this server!');
                }
                
                if (memberToKick.roles.highest.position >= message.member.roles.highest.position) {
                    return message.reply('❌ You cannot kick this user!');
                }
                
                await memberToKick.kick(kickReason);
                message.reply(`✅ Successfully kicked ${userToKick.tag} | Reason: ${kickReason}`);
                break;

            case 'purge':
                if (!isStaff(message.member)) {
                    return message.reply('❌ This command is staff only!');
                }
                if (!message.member.permissions.has(PermissionFlagsBits.ManageMessages)) {
                    return message.reply('❌ You don\'t have permission to manage messages!');
                }
                
                const amount = parseInt(args[0]);
                if (isNaN(amount) || amount < 1 || amount > 100) {
                    return message.reply('❌ Please provide a number between 1 and 100!');
                }
                
                await message.channel.bulkDelete(amount + 1, true);
                const purgeMsg = await message.channel.send(`✅ Successfully deleted ${amount} messages!`);
                setTimeout(() => purgeMsg.delete().catch(() => {}), 3000);
                break;

            case 'blacklist':
                if (!isStaff(message.member)) {
                    return message.reply('❌ This command is staff only!');
                }
                
                const userToBlacklist = message.mentions.users.first();
                if (!userToBlacklist) {
                    return message.reply('❌ Please mention a user to blacklist!');
                }
                
                if (data.blacklist.includes(userToBlacklist.id)) {
                    return message.reply('❌ This user is already blacklisted!');
                }
                
                data.blacklist.push(userToBlacklist.id);
                saveData();
                message.reply(`✅ Successfully blacklisted ${userToBlacklist.tag}`);
                break;

            case 'unblacklist':
                if (!isStaff(message.member)) {
                    return message.reply('❌ This command is staff only!');
                }
                
                const userToUnblacklist = message.mentions.users.first();
                if (!userToUnblacklist) {
                    return message.reply('❌ Please mention a user to unblacklist!');
                }
                
                if (!data.blacklist.includes(userToUnblacklist.id)) {
                    return message.reply('❌ This user is not blacklisted!');
                }
                
                data.blacklist = data.blacklist.filter(id => id !== userToUnblacklist.id);
                saveData();
                message.reply(`✅ Successfully unblacklisted ${userToUnblacklist.tag}`);
                break;

            case 'lockchannel':
                if (!isStaff(message.member)) {
                    return message.reply('❌ This command is staff only!');
                }
                if (!message.member.permissions.has(PermissionFlagsBits.ManageChannels)) {
                    return message.reply('❌ You don\'t have permission to manage channels!');
                }
                
                await message.channel.permissionOverwrites.edit(message.guild.roles.everyone, {
                    SendMessages: false
                });
                message.reply('🔒 Channel locked!');
                break;

            case 'unlockchannel':
                if (!isStaff(message.member)) {
                    return message.reply('❌ This command is staff only!');
                }
                if (!message.member.permissions.has(PermissionFlagsBits.ManageChannels)) {
                    return message.reply('❌ You don\'t have permission to manage channels!');
                }
                
                await message.channel.permissionOverwrites.edit(message.guild.roles.everyone, {
                    SendMessages: null
                });
                message.reply('🔓 Channel unlocked!');
                break;

            case 'lockallchannels':
                if (!isStaff(message.member)) {
                    return message.reply('❌ This command is staff only!');
                }
                if (!message.member.permissions.has(PermissionFlagsBits.ManageChannels)) {
                    return message.reply('❌ You don\'t have permission to manage channels!');
                }
                
                const channels = message.guild.channels.cache.filter(ch => ch.isTextBased());
                let lockedCount = 0;
                
                for (const [id, channel] of channels) {
                    try {
                        await channel.permissionOverwrites.edit(message.guild.roles.everyone, {
                            SendMessages: false
                        });
                        lockedCount++;
                    } catch (err) {
                        console.error(`Failed to lock ${channel.name}:`, err);
                    }
                }
                
                message.reply(`🔒 Locked ${lockedCount} channels!`);
                break;

            case 'unlockallchannels':
                if (!isStaff(message.member)) {
                    return message.reply('❌ This command is staff only!');
                }
                if (!message.member.permissions.has(PermissionFlagsBits.ManageChannels)) {
                    return message.reply('❌ You don\'t have permission to manage channels!');
                }
                
                const channelsToUnlock = message.guild.channels.cache.filter(ch => ch.isTextBased());
                let unlockedCount = 0;
                
                for (const [id, channel] of channelsToUnlock) {
                    try {
                        await channel.permissionOverwrites.edit(message.guild.roles.everyone, {
                            SendMessages: null
                        });
                        unlockedCount++;
                    } catch (err) {
                        console.error(`Failed to unlock ${channel.name}:`, err);
                    }
                }
                
                message.reply(`🔓 Unlocked ${unlockedCount} channels!`);
                break;

            case 'autorole':
                if (!isStaff(message.member)) {
                    return message.reply('❌ This command is staff only!');
                }
                
                const subCommand = args[0]?.toLowerCase();
                
                if (subCommand === 'set') {
                    const role = message.mentions.roles.first();
                    if (!role) {
                        return message.reply('❌ Please mention a role to set as autorole!');
                    }
                    
                    data.autorole = role.id;
                    saveData();
                    message.reply(`✅ Autorole set to ${role.name}`);
                } else if (subCommand === 'remove') {
                    data.autorole = null;
                    saveData();
                    message.reply('✅ Autorole removed!');
                } else if (subCommand === 'status') {
                    if (data.autorole) {
                        const role = message.guild.roles.cache.get(data.autorole);
                        message.reply(`📋 Current autorole: ${role ? role.name : 'Role not found'}`);
                    } else {
                        message.reply('📋 No autorole is currently set.');
                    }
                } else {
                    message.reply('❌ Usage: !autorole <set/remove/status> [@role]');
                }
                break;

            case 'setwelcome':
                if (!isStaff(message.member)) {
                    return message.reply('❌ This command is staff only!');
                }
                
                const welcomeChannel = message.mentions.channels.first();
                if (!welcomeChannel) {
                    return message.reply('❌ Please mention a channel for welcome messages!');
                }
                
                if (!welcomeChannel.isTextBased()) {
                    return message.reply('❌ Please mention a text channel!');
                }
                
                data.welcomeChannel = welcomeChannel.id;
                saveData();
                message.reply(`✅ Welcome channel set to ${welcomeChannel}`);
                break;

            case '8ball':
                const question = args.join(' ');
                if (!question) {
                    return message.reply('❌ Please ask a question!');
                }
                
                const responses = [
                    'Yes, definitely!', 'It is certain.', 'Without a doubt.',
                    'Most likely.', 'Outlook good.', 'Yes.',
                    'Reply hazy, try again.', 'Ask again later.', 'Cannot predict now.',
                    'Don\'t count on it.', 'My reply is no.', 'Outlook not so good.',
                    'Very doubtful.', 'No way!', 'Absolutely not!'
                ];
                
                const answer = responses[Math.floor(Math.random() * responses.length)];
                message.reply(`🎱 ${answer}`);
                break;

            case 'meme':
                try {
                    const response = await axios.get('https://meme-api.com/gimme');
                    const memeEmbed = new EmbedBuilder()
                        .setTitle(response.data.title)
                        .setImage(response.data.url)
                        .setColor('#FF6B6B')
                        .setFooter({ text: `👍 ${response.data.ups} | r/${response.data.subreddit}` });
                    
                    message.reply({ embeds: [memeEmbed] });
                } catch (error) {
                    message.reply('❌ Failed to fetch a meme. Try again later!');
                }
                break;

            case 'joke':
                try {
                    const response = await axios.get('https://official-joke-api.appspot.com/random_joke');
                    const jokeEmbed = new EmbedBuilder()
                        .setTitle('😂 Here\'s a joke!')
                        .setDescription(`${response.data.setup}\n\n||${response.data.punchline}||`)
                        .setColor('#FFA500');
                    
                    message.reply({ embeds: [jokeEmbed] });
                } catch (error) {
                    message.reply('❌ Failed to fetch a joke. Try again later!');
                }
                break;

            case 'trivia':
                try {
                    const response = await axios.get('https://opentdb.com/api.php?amount=1&type=multiple');
                    const triviaData = response.data.results[0];
                    
                    const allAnswers = [...triviaData.incorrect_answers, triviaData.correct_answer]
                        .sort(() => Math.random() - 0.5);
                    
                    const triviaEmbed = new EmbedBuilder()
                        .setTitle('🧠 Trivia Question')
                        .setDescription(decodeHTML(triviaData.question))
                        .addFields(
                            { name: 'Category', value: triviaData.category, inline: true },
                            { name: 'Difficulty', value: triviaData.difficulty, inline: true },
                            { name: 'Answers', value: allAnswers.map((ans, i) => `${i + 1}. ${decodeHTML(ans)}`).join('\n') }
                        )
                        .setColor('#4169E1')
                        .setFooter({ text: `Answer: ${decodeHTML(triviaData.correct_answer)}` });
                    
                    message.reply({ embeds: [triviaEmbed] });
                } catch (error) {
                    message.reply('❌ Failed to fetch trivia. Try again later!');
                }
                break;

            case 'userinfo':
                const targetUser = message.mentions.users.first() || message.author;
                
                let targetMember;
                try {
                    targetMember = await message.guild.members.fetch(targetUser.id);
                } catch (error) {
                    return message.reply('❌ User not found in this server!');
                }
                
                const userEmbed = new EmbedBuilder()
                    .setTitle(`User Info - ${targetUser.tag}`)
                    .setThumbnail(targetUser.displayAvatarURL({ dynamic: true }))
                    .addFields(
                        { name: 'ID', value: targetUser.id, inline: true },
                        { name: 'Joined Server', value: targetMember.joinedAt ? targetMember.joinedAt.toDateString() : 'Unknown', inline: true },
                        { name: 'Account Created', value: targetUser.createdAt.toDateString(), inline: true },
                        { name: 'Roles', value: targetMember.roles.cache.map(r => r.name).join(', ') || 'None' }
                    )
                    .setColor('#9B59B6');
                
                message.reply({ embeds: [userEmbed] });
                break;

            case 'serverinfo':
                const serverEmbed = new EmbedBuilder()
                    .setTitle(`Server Info - ${message.guild.name}`)
                    .setThumbnail(message.guild.iconURL({ dynamic: true }))
                    .addFields(
                        { name: 'Server ID', value: message.guild.id, inline: true },
                        { name: 'Owner', value: `<@${message.guild.ownerId}>`, inline: true },
                        { name: 'Created', value: message.guild.createdAt.toDateString(), inline: true },
                        { name: 'Members', value: message.guild.memberCount.toString(), inline: true },
                        { name: 'Channels', value: message.guild.channels.cache.size.toString(), inline: true },
                        { name: 'Roles', value: message.guild.roles.cache.size.toString(), inline: true }
                    )
                    .setColor('#3498DB');
                
                message.reply({ embeds: [serverEmbed] });
                break;

            case 'avatar':
                const avatarUser = message.mentions.users.first() || message.author;
                const avatarEmbed = new EmbedBuilder()
                    .setTitle(`${avatarUser.tag}'s Avatar`)
                    .setImage(avatarUser.displayAvatarURL({ dynamic: true, size: 512 }))
                    .setColor('#E91E63');
                
                message.reply({ embeds: [avatarEmbed] });
                break;

            case 'help':
                const helpEmbed = new EmbedBuilder()
                    .setTitle('📚 Velocity Bot Commands')
                    .setDescription('Here are all the available commands:')
                    .addFields(
                        {
                            name: '🛡️ Protection Commands (Staff Only)',
                            value: '`!ban @user [reason]` - Ban a user\n' +
                                   '`!kick @user [reason]` - Kick a user\n' +
                                   '`!purge <amount>` - Delete messages (1-100)\n' +
                                   '`!blacklist @user` - Blacklist from bot\n' +
                                   '`!unblacklist @user` - Remove blacklist\n' +
                                   '`!lockchannel` - Lock current channel\n' +
                                   '`!unlockchannel` - Unlock current channel\n' +
                                   '`!lockallchannels` - Lock all channels\n' +
                                   '`!unlockallchannels` - Unlock all channels\n' +
                                   '`!autorole set/remove/status @role` - Manage autorole\n' +
                                   '`!setwelcome #channel` - Set welcome channel'
                        },
                        {
                            name: '🎮 Fun Commands',
                            value: '`!8ball <question>` - Ask the magic 8-ball\n' +
                                   '`!meme` - Get a random meme\n' +
                                   '`!joke` - Get a random joke\n' +
                                   '`!trivia` - Get a trivia question'
                        },
                        {
                            name: 'ℹ️ Info Commands',
                            value: '`!userinfo [@user]` - Get user information\n' +
                                   '`!serverinfo` - Get server information\n' +
                                   '`!avatar [@user]` - Get user avatar\n' +
                                   '`!help` - Show this message'
                        }
                    )
                    .setColor('#00FF00')
                    .setFooter({ text: 'Velocity Bot | Use !command to execute' });
                
                message.reply({ embeds: [helpEmbed] });
                break;

            default:
                break;
        }
    } catch (error) {
        console.error('Command error:', error);
        message.reply('❌ An error occurred while executing the command!');
    }
});

function decodeHTML(html) {
    const entities = {
        '&quot;': '"',
        '&#039;': "'",
        '&amp;': '&',
        '&lt;': '<',
        '&gt;': '>',
    };
    return html.replace(/&[#\w]+;/g, entity => entities[entity] || entity);
}

client.login(process.env.DISCORD_BOT_TOKEN || config.token);
