# youtube-channel-scraper

```jsx
require("dotenv").config();
const APIKey = process.env.API_KEY;
const axios = require("axios");
const fs = require("fs");
const path = require("path");
const xlsx = require("xlsx");

async function getChannelId(channelName) {
  try {
    const response = await axios.get(
      "https://www.googleapis.com/youtube/v3/search",
      {
        params: {
          part: "snippet",
          q: channelName,
          type: "channel",
          key: APIKey,
        },
      }
    );

    if (response.data.items.length > 0) {
      const channel = response.data.items[0].id;
      return channel.channelId;
    }
  } catch (error) {
    console.error("Error fetching channel id:");
    console.log(error.message);
  }
}

async function getChannelInfo(channelId) {
  try {
    const response = await axios.get(
      `https://www.googleapis.com/youtube/v3/channels`,
      {
        params: {
          part: "snippet,statistics,contentDetails",
          id: channelId,
          key: APIKey,
        },
      }
    );
    return response.data.items[0];
  } catch (error) {
    console.error("Error fetching channel data:");
    console.log(error.message);
  }
}

async function getVideoIdsFromChannel(channelId) {
  const YOUTUBE_API_URL = "https://www.googleapis.com/youtube/v3/";
  const videoIds = [];
  let nextPageToken = "";
  try {
    do {
      const response = await axios.get(`${YOUTUBE_API_URL}search`, {
        params: {
          key: APIKey,
          channelId,
          part: "id",
          maxResults: 50,
          type: "sh",
          pageToken: nextPageToken,
        },
      });
      const items = response.data.items;
      items.forEach((item) => {
        videoIds.push(item.id.videoId);
      });
      nextPageToken = response.data.nextPageToken || "";
    } while (nextPageToken);
    return videoIds;
  } catch (error) {
    console.error("Error fetching video ids:");
    console.log(error.message);
  }
}

// 동영상 정보 가져오기
async function getVideoDetails(videoId) {
  try {
    const response = await axios.get(
      "https://www.googleapis.com/youtube/v3/videos",
      {
        params: {
          part: "snippet,statistics,contentDetails",
          id: videoId,
          key: APIKey,
        },
      }
    );
    return response.data.items[0]; // 첫 번째 결과
  } catch (error) {
    console.error("Error fetching video detail:");
    console.log(error.message);
  }
}

// 동영상 정보 가져오기
async function getVideoReplies(videoId) {
  try {
    const response = await axios.get(
      "https://www.googleapis.com/youtube/v3/commentThreads",
      {
        params: {
          part: "snippet,replies",
          videoId: videoId,
          key: APIKey,
          maxResults: 10,
          order: "relevance", // 인기순으로 댓글 가져오기
        },
      }
    );

    // const video = response.data.items[0]; // 첫 번째 결과
    return response.data; // 첫 번째 결과
  } catch (error) {
    console.error("Error fetching video replies:");
    console.log(error.message);
  }
}

async function scrape(channelName) {
  // Get Info
  const channelId = await getChannelId(channelName);
  const channelInfo = await getChannelInfo(channelId);
  const videoIds = await getVideoIdsFromChannel(channelId);
  console.log(`Total Video Ids: ${videoIds.length}`);

  // Writing Video Data
  const videoDetails = [];
  for (const videoId of videoIds) {
    try {
      const videoDetail = await getVideoDetails(videoId);
      const replies = await getVideoReplies(videoId);
      videoDetails.push({
        video: videoDetail,
        replies,
      });

      // 각 비디오에 대해 폴더 경로 만들기
      const videoFolderPath = path.join(
        __dirname,
        "channels",
        channelName,
        "videos",
        videoId
      );

      // 폴더가 없으면 생성`
      if (!fs.existsSync(videoFolderPath)) {
        fs.mkdirSync(videoFolderPath, { recursive: true });
      }

      // video.json 파일 저장
      fs.writeFileSync(
        path.join(videoFolderPath, "video.json"),
        JSON.stringify(videoDetail, null, 2),
        "utf8"
      );

      // replies.json 파일 저장
      fs.writeFileSync(
        path.join(videoFolderPath, "replies.json"),
        JSON.stringify(replies, null, 2),
        "utf8"
      );
    } catch (err) {
      console.error("Error writing video & replies data");
      console.log(err.message);
    }
  }

  // Writing Enitre Data
  try {
    const data = {
      channel: channelInfo,
      videos: videoDetails,
    };
    const videoFolderPath = path.join(__dirname, "channels", channelName);

    // 폴더가 없으면 생성
    if (!fs.existsSync(videoFolderPath)) {
      fs.mkdirSync(videoFolderPath, { recursive: true });
    }
    fs.writeFileSync(
      path.join(videoFolderPath, `${channelName}.json`),
      JSON.stringify(data, null, 2),
      "utf8"
    );
    return data;
  } catch (err) {
    console.error("Error writing to file:");
    console.log(err.message);
  }
}

function createExcel(channelName, data) {
  const channelSnippet = data.channel.snippet;
  const channalStats = data.channel.statistics;

  // 채널 데이터 준비
  const channelData = [
    ["Channel Info", "Details"],
    ["Title", channelSnippet ? channelSnippet.title : "N/A"],
    ["Description", channelSnippet ? channelSnippet.description : "N/A"],
    ["Custom URL", channelSnippet.customUrl || "N/A"],
    ["Published At", channelSnippet.publishedAt || "N/A"],
    ["View Count", channalStats.viewCount || "N/A"],
    ["Subscriber Count", channalStats.subscriberCount || "N/A"],
    ["Video Count", channalStats.videoCount || "N/A"],
  ];

  // 비디오 데이터 준비
  const videoData = [
    [
      "Video ID",
      "Title",
      "Published At",
      "View Count",
      "Like Count",
      "Comment Count",
    ],
    ...(data.videos ? data.videos : []).map((video) => {
      // If 'video' exists, then access its properties directly, else fallback to "N/A"
      const videoDetails = video.video || {}; // Assuming video is under 'video'
      const snippet = videoDetails.snippet || {}; // Assuming snippet is inside video
      const statistics = videoDetails.statistics || {}; // Assuming statistics is inside video

      return [
        videoDetails.id || "N/A", // video ID
        snippet.title || "N/A", // video title
        snippet.publishedAt || "N/A", // video published date
        statistics.viewCount || "N/A", // view count
        statistics.likeCount || "N/A", // like count
        statistics.commentCount || "N/A", // comment count
      ];
    }),
  ];

  // 새 워크북 생성
  const wb = xlsx.utils.book_new();

  // 데이터 시트 추가
  const channelSheet = xlsx.utils.aoa_to_sheet(channelData);
  const videoSheet = xlsx.utils.aoa_to_sheet(videoData);

  // 워크북에 시트 추가
  xlsx.utils.book_append_sheet(wb, channelSheet, "Channel Info");
  xlsx.utils.book_append_sheet(wb, videoSheet, "Videos");

  // 워크북을 파일로 저장
  const videoFolderPath = path.join(__dirname, "channels", channelName);

  // 폴더가 없으면 생성
  if (!fs.existsSync(videoFolderPath)) {
    fs.mkdirSync(videoFolderPath, { recursive: true });
  }

  // 워크북을 파일로 저장 (xlsx.writeFile을 사용)
  const filePath = path.join(videoFolderPath, `${channelName}.xlsx`);
  xlsx.writeFile(wb, filePath); // wb는 워크북 객체
  console.log("Excel file has been created at:", filePath);
}

async function main() {
  const channelName = "여우튜브";
  const data = await scrape(channelName);
  createExcel(data);
}

main();
```
# MacApp-BenchmarkYoutube
# MacApp-BenchmarkYoutube
