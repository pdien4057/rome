# rome
async function sendReplyMessage(channelId, token, agent) {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), DISCORD_API_TIMEOUT);

  try {
    // Kiểm tra dữ liệu chatContents
    if (chatContents.length === 0) {
      console.log(chalk.yellow(`[!] File data.txt trống.`));
      return null;
    }

    // Lấy 50 tin nhắn gần nhất
    const messagesRes = await fetch(`https://discord.com/api/v9/channels/${channelId}/messages?limit=50`, {
      headers: { 'Authorization': token },
      agent,
      signal: controller.signal
    });

    if (!messagesRes.ok) {
      console.log(chalk.red(`[x] Lỗi lấy tin nhắn: ${messagesRes.status} ${await messagesRes.text()}`));
      return null;
    }

    const messages = await messagesRes.json();

    // Lọc tin nhắn hợp lệ
    const validMessages = messages.filter(msg => {
      const isMentioningMe = CHECK_MENTIONS &&
        Array.isArray(msg.mentions) &&
        msg.mentions.some(mention => mention.id === CLIENT_USER_ID);

      return (
        !msg.author.bot &&
        !excludedUserIds.includes(msg.author.id) &&
        (msg.content.length > 5 || isMentioningMe)
      );
    });

    if (validMessages.length === 0) {
      console.log(chalk.yellow(`[!] Không tìm thấy tin nhắn hợp lệ.`));
      return null;
    }

    const mentionedMessages = validMessages.filter(msg =>
      Array.isArray(msg.mentions) &&
      msg.mentions.some(mention => mention.id === CLIENT_USER_ID)
    );

    let targetMsg;
    let replyContent;

    if (mentionedMessages.length > 0) {
      const mentionMsg = mentionedMessages[0];

      if (Math.random() < 0.25 && validMessages.length >= 2) {
        const otherMessages = validMessages.filter(msg => msg.author.id !== mentionMsg.author.id);
        if (otherMessages.length === 0) return null;

        const randomMsg = otherMessages[Math.floor(Math.random() * otherMessages.length)];
        const randomLine = chatContents[Math.floor(Math.random() * chatContents.length)];

        // Tạo reply từ DeepSeek
        const deepSeekReply = await generateDeepSeekReply(mentionMsg.content, [
          { role: "assistant", content: "Short, natural answer" }
        ]);

        replyContent = [
          `${randomLine}`,                                 // Dòng 1: random người khác
          `<@${mentionMsg.author.id}> ${deepSeekReply}`    // Dòng 2: phản hồi người mention
        ].join('\n');

        targetMsg = validMessages.length > 1 ? validMessages[1] : mentionMsg;
      } else {
        // Trả lời người mention bằng DeepSeek
        replyContent = await generateDeepSeekReply(mentionMsg.content, [
          { role: "assistant", content: "Short, natural answer" }
        ]);
        targetMsg = mentionMsg;
      }
    } else {
      // Trường hợp không ai mention
      targetMsg = validMessages.length > 1 ? validMessages[1] : validMessages[0];
      const randomLine = chatContents[Math.floor(Math.random() * chatContents.length)];
      replyContent = `${randomLine}`;
    }

    // Gửi tin nhắn
    const res = await fetch(`https://discord.com/api/v9/channels/${channelId}/messages`, {
      method: 'POST',
      headers: {
        'Authorization': token,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        content: replyContent,
        message_reference: {
          message_id: targetMsg.id,
          channel_id: channelId,
          guild_id: targetMsg.guild_id || undefined
        }
      }),
      agent,
      signal: controller.signal
    });

    if (res.ok) {
      const msgData = await res.json();
      console.log(chalk.green(`[✔] Đã reply: "${replyContent}"`));

      if (deleteOption) {
        await new Promise(resolve => setTimeout(resolve, waktuHapus));
        await deleteMessage(channelId, msgData.id, token, agent);
      }
      return msgData.id;
    }
  } catch (err) {
    if (err.name === 'AbortError') {
      console.log(chalk.red(`[x] Request bị timeout sau ${DISCORD_API_TIMEOUT}ms`));
    } else {
      console.log(chalk.red(`[x] Lỗi khi reply: ${err.message}`));
    }
  } finally {
    clearTimeout(timeoutId);
  }

  return null;
}

