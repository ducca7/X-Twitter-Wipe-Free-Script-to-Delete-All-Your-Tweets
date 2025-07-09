# X-Twitter-Wipe-Free-Script-to-Delete-All-Your-Tweets
Seeking a clean slate or enhanced privacy? I got you fam! Easily remove all your tweets from X (formerly Twitter) with this free, open-source script. No paid services, no API keys required—just run the script in your browser console and wipe your entire tweet history. 

1. Log into your (X) Twitter account on desktop (tested in Firefox and Chrome)
2. Go to your X profile with replies (https://x.com/YOURUSERNAME/with_replies)
3. Open the browser console (F12 or cmd+option+i)
4. Paste the whole script into the console and press Enter
5. Wait for all your Tweets to vanish
6. If it gets stuck, refresh the page and repaste the script in the browser console again
7. Repeat after me: _"F*ck Elon Musk!"_
8. Enjoy life!


```
const deleteAllTweets = async () => {
  const processed = new Set();
  const stats = { deleted: 0, failed: 0, retries: 0 };
  
  // Flexible selectors with fallbacks
  const selectors = {
    tweet: '[data-testid="tweet"]',
    caret: '[data-testid="caret"]',
    menuItem: '[role="menuitem"]',
    deleteConfirm: '[data-testid="confirmationSheetConfirm"]',
    unretweet: '[data-testid="unretweet"]',
    unretweetConfirm: '[data-testid="unretweetConfirm"]'
  };

  // Fallback selectors if main ones fail
  const fallbackSelectors = {
    caret: 'button[aria-haspopup="menu"]',
    deleteConfirm: 'button:contains("Delete")',
    menuItem: '[data-testid*="menu"]'
  };

  const delay = ms => {
    const jitter = ms * 0.25;
    const actual = ms + (Math.random() * jitter * 2 - jitter);
    return new Promise(resolve => setTimeout(resolve, actual));
  };

  // Enhanced selector with fallbacks
  const getButtons = () => {
    let buttons = Array.from(document.querySelectorAll(`${selectors.tweet} ${selectors.caret}`));
    
    // Fallback if no buttons found
    if (buttons.length === 0) {
      buttons = Array.from(document.querySelectorAll(`${selectors.tweet} ${fallbackSelectors.caret}`));
    }
    
    return buttons.filter(b => !processed.has(b));
  };

  const scrollToEnd = () =>
    window.scrollTo({ top: document.body.scrollHeight, behavior: 'smooth' });

  // Rate limit detection
  const detectRateLimit = () => {
    const pageText = document.body.innerText.toLowerCase();
    return pageText.includes('rate limit') || pageText.includes('try again later');
  };

  // Retry logic with exponential backoff
  const attemptDelete = async (button, maxRetries = 3) => {
    for (let attempt = 0; attempt < maxRetries; attempt++) {
      try {
        processed.add(button);
        
        // Check for rate limiting
        if (detectRateLimit()) {
          console.log('⚠️ Rate limit detected, waiting 5 minutes...');
          await delay(5 * 60 * 1000);
          stats.retries++;
          continue;
        }
        
        button.click();
        await delay(250);

        const menuItems = Array.from(document.querySelectorAll(selectors.menuItem));
        const deleteOption = menuItems.find(item => item.textContent.includes('Delete'));

        if (deleteOption) {
          deleteOption.click();
          await delay(250);
          
          let confirm = document.querySelector(selectors.deleteConfirm);
          // Fallback for confirm button
          if (!confirm) {
            const buttons = Array.from(document.querySelectorAll('button'));
            confirm = buttons.find(btn => btn.textContent.toLowerCase().includes('delete'));
          }
          
          if (confirm) {
            confirm.click();
            await delay(2000);
            stats.deleted++;
            console.log(`✅ Deleted tweet #${stats.deleted}`);
            return true;
          }
        }

        const tweet = button.closest(selectors.tweet);
        const unretweet = tweet?.querySelector(selectors.unretweet);
        if (unretweet) {
          unretweet.click();
          await delay(250);
          const confirm = document.querySelector(selectors.unretweetConfirm);
          if (confirm) {
            confirm.click();
            await delay(2000);
            stats.deleted++;
            console.log(`🔄 Unretweeted #${stats.deleted}`);
            return true;
          }
        }
        
        // If we get here, the attempt failed
        if (attempt < maxRetries - 1) {
          const backoffDelay = 1000 * Math.pow(2, attempt);
          console.log(`🔄 Retry ${attempt + 1} in ${backoffDelay}ms`);
          await delay(backoffDelay);
          stats.retries++;
        }
        
      } catch (err) {
        console.error(`❌ Error on attempt ${attempt + 1}:`, err.message);
        if (attempt < maxRetries - 1) {
          const backoffDelay = 1000 * Math.pow(2, attempt);
          await delay(backoffDelay);
          stats.retries++;
        }
      }
    }
    
    stats.failed++;
    console.log(`❌ Failed to delete tweet after ${maxRetries} attempts`);
    return false;
  };

  console.log('🚀 Starting tweet deletion...');
  let consecutiveEmpty = 0;
  
  while (true) {
    const buttons = getButtons();
    
    if (!buttons.length) {
      scrollToEnd();
      await delay(5000);
      
      if (!getButtons().length) {
        consecutiveEmpty++;
        if (consecutiveEmpty >= 3) {
          console.log('✅ No more tweets found after 3 attempts');
          break;
        }
        console.log(`📜 No tweets found, attempt ${consecutiveEmpty}/3`);
        await delay(3000);
        continue;
      }
      consecutiveEmpty = 0;
      continue;
    }
    
    consecutiveEmpty = 0;
    console.log(`🎯 Processing ${buttons.length} tweets...`);

    for (const button of buttons) {
      await attemptDelete(button);
      await delay(500); // brief cooldown between deletions
    }

    scrollToEnd();
    await delay(2000); // help load more tweets
  }

  // Final stats
  console.log('\n🎉 Deletion complete!');
  console.log(`✅ Successfully deleted: ${stats.deleted}`);
  console.log(`❌ Failed attempts: ${stats.failed}`);
  console.log(`🔄 Total retries: ${stats.retries}`);
  
  if (stats.deleted + stats.failed > 0) {
    const successRate = ((stats.deleted / (stats.deleted + stats.failed)) * 100).toFixed(1);
    console.log(`📊 Success rate: ${successRate}%`);
  }
};

deleteAllTweets().catch(err => console.error('Script failed:', err));
