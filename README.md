# empiachannel
Video Player Channel
<?php
/*
Plugin Name:  Video Scheduler 
Description:  Allows scheduling of videos on pages. 
Version:      1
Author:       Alexander Hansen <alex@alex-hansen.com> 
*/

define('VIDEO_SCHEDULER_PATH', plugin_dir_path(__FILE__)); 
$page_slug = trim( $_SERVER["REQUEST_URI"] , '/' );
$is_home = is_front_page() ? 'home' : 'nothome';
define('WP_DEBUG', true);

// The code for the admin interface.
function video_scheduler_register_settings() {
   add_option( 'video_scheduler_codes', 'Video codes.');
   register_setting( 'video_scheduler_options_group', 'video_scheduler_codes', 'video_scheduler_callback' );
   add_option( 'video_scheduler_pages', 'Pages where the scheduled videos appear.');
   register_setting( 'video_scheduler_options_group', 'video_scheduler_pages', 'video_scheduler_callback' ); } add_action( 'admin_init', 'video_scheduler_register_settings' );

function video_scheduler_register_options_page() {
  add_options_page('Video Scheduler', 'Video Scheduler', 'manage_options', 'video_scheduler', 'video_scheduler_options_page');
}
add_action('admin_menu', 'video_scheduler_register_options_page');

function video_scheduler_options_page() {
?>
  <div>
  <?php screen_icon(); ?>
  <h2>Video Scheduler</h2>
  <p> A plugin by Alex Hansen (<a href="mailto:alex@alex-hansen.com">alex@alex-hansen.com</a>).
  <form method="post" action="options.php">
  <?php settings_fields( 'video_scheduler_options_group' ); ?>
  <p>Please enter the video codes and times here, one per line, with a comma followed by their time.</p>
  <table>
  <tr valign="top">
  <th><label for="video_scheduler_codes">Video Codes</label></th>
  <th> Current Schedule </th>
</tr>
<tr>
  <td><textarea style="height: 3900px; width: 300px;" type="textarea" id="video_scheduler_codes" name="video_scheduler_codes"><?php echo get_option('video_scheduler_codes'); ?> </textarea></td>
<td> <?php echo get_video_and_length_html(); ?> </td>
</tr>

<tr> <th> Pages where the videos should appear, separated by a commas: </th></tr>
<tr> <td> <input style="width: 400px;" type="text" id="video_scheduler_pages" name="video_scheduler_pages" value="<?php echo get_option('video_scheduler_pages'); ?>" /> </td> </tr>
  </table>
  <?php  submit_button(); ?>
  </form>
  </div>
<?php
}
// [bartag foo="foo-value"]
// [
function video_shortcode_func( $atts ) {
    $a = shortcode_atts( array(
        'width' => '560',
        'height' => '315',
    ), $atts );

    $width = $a['width'];
    $height = $a['height'];

    return get_video_url($width, $height);
}
add_shortcode( 'empiachannel', 'video_shortcode_func' );



// All of the pages where the map should appear are in this array.
$pages = get_option('video_scheduler_pages');
$pages_where_the_videos_should_appear = explode(",", $pages);

//if (in_array($page_slug, $pages_where_the_videos_should_appear)) {
//    add_action('the_content', 'add_videos');
//}

function format_seconds_as_time($secs) {
	$seconds = $secs % 60;
	$minutes = floor(($secs % 3600) / 60);
	$hours = floor($secs / 3600);

	return $hours . " : " . sprintf('%02d', $minutes) . " : " . sprintf('%02d', $seconds);
}

function get_video_and_length_html() {
	$videos = get_videos_from_option();
	$lengths = get_lengths_from_option();
	$formatted_lengths = array();
	$start_times = array();
	for ($i = 0; $i < count($lengths); $i++) {
		$minutes = floor($lengths[$i] / 60);
		$seconds = $lengths[$i] % 60;
		$formatted_lengths[$i] = sprintf('%02d', $minutes) . " : " . sprintf('%02d', $seconds);
		if ($i == 0) {
			$start_times[$i] = 0; 
			$start_times[$i + 1] = $lengths[$i];
		}
		if ($i == 1) {}
		else {
			$start_times[$i] = $start_times[$i-1] + $lengths[$i - 1];
		}
	}

	$output = "<table><col width=\"150\"><col width=\"150\"><col width=\"150\"><tr><th> Video Code </th> <th> Length </th> <th> Estimated Start Time </th> </tr>";
	for ($i = 0; $i < count($formatted_lengths); $i++) {
		$output .= "<tr><td>";
		$output .= $videos[$i] . "</td><td>";
		$output .= $formatted_lengths[$i] . "</td><td>";
		$output .= format_seconds_as_time($start_times[$i]); 
		$output .= "</td></tr>";
		$output .= "</table";
	}
	return $output;

}

function get_videos_from_option() {
	$video_option = get_option('video_scheduler_codes');
	$videos_and_lengths = explode("\n", $video_option); 
	$videos = array();
	for ($i = 0; $i < count($videos_and_lengths); $i++) {
		$split = explode(',', $videos_and_lengths[$i]);
		$videos[$i] = trim($split[0]);
	}
	return $videos;
}

function get_lengths_from_option() {

	$video_option = get_option('video_scheduler_codes');
	$videos_and_lengths = explode("\n", $video_option); 
	$lengths = array();
	for ($i = 0; $i < count($videos_and_lengths); $i++) {
		$split = explode(',', $videos_and_lengths[$i]);
		$lengths[$i] = trim($split[1]);
	}
	return $lengths;


}

// The code to add videos to the page.
function get_video_url($width, $height) {
	$videos = get_videos_from_option();
	$lengths = get_lengths_from_option();

	$time_since_midnight = time() - strtotime("today");


	// Time in seconds of videos, in order
	$current_video_index = -1;
	$time_counter = 0;

	for ($i = 0; $i < count($lengths); $i++) {
		if ($time_counter < $time_since_midnight) {
			$time_counter = $time_counter + $lengths[$i];
			$current_video_index++;
		}
	}
	if ($time_counter < $time_since_midnight) {
//		if get_option('display_no_programming') echo "No programming at this time (" . $time_since_midnight . ") . Displaying the last scheduled video. ";
	}
//	$content .= "time: " . $time;
	$overflow_seconds = 0;
	if ($time_counter > $time_since_midnight) {
		$overflow_seconds = $time_counter - $time_since_midnight;
	}
	if (strcmp("dead_time", trim($videos[$current_video_index])) == 0) {
		return "No videos scheduled for this time.";
	}
	else {
//		echo 'current video and present time/counter time: ' . trim($videos[$current_video_index]) . $time_since_midnight . ' ' . $time_counter;
		return '<iframe width="' . $width . '" height="' . $height . '" src="https://www.youtube.com/embed/' . trim($videos[$current_video_index]) . '?controls=0&modestbranding=1&disablekb=1&showinfo=0&rel=0&autoplay=1&start=' . $overflow_seconds . '" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>';
	}
}


