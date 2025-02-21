import discord
import random
import re
import yt_dlp as youtube_dl
from discord.ext import commands
import os       

# 디스코드 봇 토큰 직접 설정 (테스트 목적으로만 사용)
TOKEN = "토큰"

# FFmpeg 실행 파일 경로 설정
FFMPEG_PATH = "C:/ffmpeg/bin/ffmpeg.exe"

# Intents 설정
intents = discord.Intents.default()
intents.message_content = True
intents.messages = True  # 메시지 관리 권한 추가
bot = commands.Bot(command_prefix='!', intents=intents)

# 재생 목록 저장
music_queue = []

# 봇 준비 완료 메시지 출력
@bot.event
async def on_ready():
    print(f'✅ {bot.user}로 로그인됨')

# 명령어 목록 출력
@bot.command()
async def 도움말(ctx):
    commands_info = (
        "📜 **명령어 목록**\n"
        "!주사위 [XdY] [성공조건(선택)] [목표값(선택)] - 주사위를 굴리고 성공 여부를 확인합니다.\n"
        "!불러오기 - 봇을 음성 채널에 참여시킵니다.\n"
        "!나가 - 봇을 음성 채널에서 나가게 합니다.\n"
        "!목록추가 [유튜브 URL] - 재생 목록에 노래를 추가합니다.\n"
        "!재생 [유튜브 URL] - 입력한 URL의 노래를 재생합니다.\n"
        "!삭제 [숫자] - 지정된 개수의 채팅을 삭제합니다. (기본 10)\n"
        "!목록보기 - 현재 재생 목록을 확인합니다.\n"
        "!목록제거 [번호] - 재생 목록에서 해당 번호의 노래를 제거합니다.\n"
        "!일시중지 - 현재 재생 중인 노래를 일시 중지합니다.\n"
        "!다시재생 - 일시 중지된 노래를 다시 재생합니다."
    )
    await ctx.send(commands_info)

# 주사위 굴림 함수
def roll_dice(dice: str, success_condition: str = None, target_value: int = None):
    match = re.fullmatch(r"(\d*)d(\d+)", dice)
    if not match:
        return "잘못된 형식입니다. 예: !주사위 2d6 >= 7"

    num_dice = int(match.group(1)) if match.group(1) else 1
    dice_sides = int(match.group(2))

    if num_dice <= 0 or dice_sides <= 0:
        return "주사위 개수와 면 수는 1 이상이어야 합니다."

    rolls = [random.randint(1, dice_sides) for _ in range(num_dice)]
    total = sum(rolls)

    result_message = f"🎲 주사위 결과: {rolls} (총합: {total})"

    if success_condition and target_value is not None:
        if success_condition == '>=':
            success = total >= target_value
        elif success_condition == '<=':
            success = total <= target_value
        elif success_condition == '>':
            success = total > target_value
        elif success_condition == '<':
            success = total < target_value
        elif success_condition in ('=', '=='):
            success = total == target_value
        else:
            return "잘못된 성공 조건입니다. 사용 가능한 조건: >=, <=, >, <, ="

        result_message += f" - {'성공' if success else '실패'}! (조건: 총합 {success_condition} {target_value})"

    return result_message

@bot.command()
@commands.cooldown(rate=3, per=5, type=commands.BucketType.user)
async def 주사위(ctx, dice: str, success_condition: str = None, target_value: int = None):
    result = roll_dice(dice, success_condition, target_value)
    await ctx.send(result)

@주사위.error
async def 주사위_error(ctx, error):
    if isinstance(error, commands.MissingRequiredArgument):
        await ctx.send("사용법: `!주사위 [주사위식] [성공조건(선택)] [목표값(선택)]` 예: `!주사위 2d6 >= 7`")
    elif isinstance(error, commands.BadArgument):
        await ctx.send("목표값은 숫자여야 합니다. 예: `!주사위 2d6 >= 7`")
    else:
        await ctx.send("오류가 발생했습니다. 다시 시도해주세요.")
        print(f"오류 발생: {error}")

# 봇이 음성 채널에 참가하는 명령어
@bot.command()
async def 불러오기(ctx):
    if ctx.author.voice:
        channel = ctx.author.voice.channel
        try:
            await channel.connect(reconnect=True, timeout=60.0)
            await ctx.send("🔊 음성 채널에 참가했습니다.")
        except discord.errors.ClientException:
            await ctx.send("🚫 이미 음성 채널에 연결되어 있습니다.")
        except Exception as e:
            await ctx.send(f"🚫 음성 채널에 연결하는 동안 오류가 발생했습니다: {e}")
    else:
        await ctx.send("🚫 먼저 음성 채널에 접속해주세요.")

# 봇이 음성 채널을 나가는 명령어
@bot.command()
async def 나가(ctx):
    if ctx.voice_client:
        try:
            await ctx.voice_client.disconnect()
            await ctx.send("👋 음성 채널에서 나갔습니다.")
        except Exception as e:
            await ctx.send(f"🚫 음성 채널에서 나가는 동안 오류가 발생했습니다: {e}")
    else:
        await ctx.send("🚫 봇이 현재 음성 채널에 없습니다.")

# 노래 추가 명령어
@bot.command()
async def 목록추가(ctx, url: str):
    ydl_opts = {
        'format': 'bestaudio/best',
        'quiet': True,
        'extract_flat': False,
        'noplaylist': True
    }
    
    try:
        with youtube_dl.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(url, download=False)
            title = info.get('title', '알 수 없는 제목')
            music_queue.append((url, title))
            await ctx.send(f'🎵 추가됨: {title} (총 {len(music_queue)}곡)')
    except Exception as e:
        await ctx.send(f'❌ 유튜브에서 정보를 가져오는 중 오류 발생: {e}')

# 유튜브 음악 재생 명령어
@bot.command()
async def 재생(ctx, url: str = None):
    if not ctx.author.voice:
        await ctx.send("🚫 먼저 음성 채널에 접속해주세요.")
        return
    
    if not ctx.voice_client:
        await ctx.invoke(불러오기)
    
    if url:
        ydl_opts = {
            'format': 'bestaudio/best',
            'quiet': True,
            'extract_flat': False,
            'noplaylist': True
        }
        try:
            with youtube_dl.YoutubeDL(ydl_opts) as ydl:
                info = ydl.extract_info(url, download=False)
                title = info.get('title', '알 수 없는 제목')
                music_queue.insert(0, (url, title))
        except Exception as e:
            await ctx.send(f'❌ 유튜브에서 정보를 가져오는 중 오류 발생: {e}')
            return

    if ctx.voice_client.is_playing() or not music_queue:
        await ctx.send("🎶 재생할 노래가 없습니다. !목록추가 [유튜브 URL]로 추가해주세요.")
        return
    
    url, title = music_queue.pop(0)
    
    ydl_opts = {
        'format': 'bestaudio/best',
        'quiet': True,
        'extract_flat': False,
        'noplaylist': True
    }
    
    try:
        with youtube_dl.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(url, download=False)
            audio_url = info['url']
    except Exception as e:
        await ctx.send(f'❌ 유튜브에서 정보를 가져오는 중 오류 발생: {e}')
        return
    
    ffmpeg_opts = {
        'before_options': '-reconnect 1 -reconnect_streamed 1 -reconnect_delay_max 5',
        'options': '-vn'
    }
    
    try:
        source = discord.FFmpegPCMAudio(audio_url, executable=FFMPEG_PATH, **ffmpeg_opts)
        ctx.voice_client.play(source, after=lambda e: bot.loop.create_task(재생(ctx)) if music_queue else None)
        await ctx.send(f'🎶 재생 중: {title}')
    except Exception as e:
        await ctx.send(f'❌ 음악 재생 중 오류 발생: {e}')

# 음악 일시중지 명령어
@bot.command()
async def 일시중지(ctx):
    if ctx.voice_client and ctx.voice_client.is_playing():
        ctx.voice_client.pause()
        await ctx.send("⏸ 음악을 일시 중지했습니다.")
    else:
        await ctx.send("🚫 현재 재생 중인 음악이 없습니다.")

# 음악 다시재생 명령어
@bot.command()
async def 다시재생(ctx):
    if ctx.voice_client and ctx.voice_client.is_paused():
        ctx.voice_client.resume()
        await ctx.send("▶ 음악을 다시 재생합니다.")
    else:
        await ctx.send("🚫 현재 일시중지된 음악이 없습니다.")

# 재생 목록에서 특정 곡을 제거하는 명령어
@bot.command()
async def 목록제거(ctx, index: int):
    if 0 < index <= len(music_queue):
        removed_song = music_queue.pop(index - 1)
        await ctx.send(f'🗑 목록에서 제거됨: {removed_song[1]}')
    else:
        await ctx.send("🚫 잘못된 인덱스입니다. 올바른 번호를 입력하세요.")

# 현재 재생 목록을 출력하는 명령어
@bot.command()
async def 목록보기(ctx):
    if not music_queue:
        await ctx.send("🎶 현재 재생 목록이 비어 있습니다.")
    else:
        queue_list = '\n'.join(f"{i+1}. {title}" for i, (url, title) in enumerate(music_queue))
        await ctx.send(f"📜 현재 재생 목록:\n{queue_list}")

# 채팅 삭제 명령어
@bot.command()
async def 삭제(ctx, amount: int = 10):
    await ctx.channel.purge(limit=amount + 1)
    await ctx.send(f'🗑 {amount}개의 메시지를 삭제했습니다.', delete_after=5)

# 봇 실행
bot.run(TOKEN)
